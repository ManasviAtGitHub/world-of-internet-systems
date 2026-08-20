# Tinder

## One-line hook
Dating is local, so Tinder sharded its entire recommendation index by geography — the world cut
into S2 map cells sized by population density ("geosharding") — and turned the swipe→match moment
into a stream-plus-cache lookup delivered over a WebSocket "nudge."

## The core problem
Every recommendation query is location-bounded: Tinder shows you people within a chosen radius
(their published cap: 100 miles), so a user in California never needs London profiles. Yet the
original design kept everyone in one Elasticsearch index. As the user base grew to tens of
millions, that single index became too big and too slow: every query scanned mostly-irrelevant
global data, latency climbed, and scaling meant buying ever more replicas. Meanwhile the write
side is brutal in its own way — on the order of a billion-plus swipes per day, each of which
might be the second half of a mutual like that must become a match, visible to both phones,
within moments.

## Architecture overview
End-to-end data flow, from opening the app to "It's a Match!":

1. **Profile & location ingestion.** Profile edits and location updates are produced to Kafka.
   Kafka partitions by user so each user's updates are processed in order — this ordering is a
   load-bearing consistency mechanism (see below). Consumers write the user's document into the
   correct **geoshard** of the search datastore.
2. **Geosharded search index.** The recommendation index is Elasticsearch, but split into shards
   that correspond to geographic regions built from Google S2 cells. Dense metros get small cells
   (S2 level ~8, roughly 22.5 mi across), sparse regions get bigger ones (level ~7, roughly 45
   mi), so each shard holds a comparable number of users. In Elasticsearch terms, each geoshard
   is an index; replicas are scaled per-shard by traffic, and shards from different time zones
   are deliberately mixed onto the same nodes so daily peaks in one longitude balance troughs in
   another.
3. **Recommendation query.** When a user opens the card stack, the recs service resolves their
   position + distance filter to the overlapping geoshard(s) — near a shard border the query fans
   out to two or three — runs the filtered query (age, preferences, already-swiped exclusions),
   then ranks candidates. Ranking evolved from the retired ELO-style desirability score
   (officially dropped by 2019) to engagement-driven models; TinVec (MLconf 2017) embedded users
   as vectors from swipe co-occurrence — people liked by the same people end up nearby in vector
   space. The Elasticsearch 8 migration added kNN/vector search and deep-learning ranking.
4. **Swipe → match pipeline.** Swipes go to a swipes service that writes them to a data stream
   (Kinesis-style). Left swipes are archived to cheap storage (S3). Right swipes are read by
   match workers, which check a **Likes cache** (Redis-style) — "has the other person already
   liked me?" If yes: create the match record (DynamoDB per public writeups), fan out to both
   users. If no: record my like in the cache and wait.
5. **Delivery: Keepalive & Nudges.** Before 2019, every open app polled the server every 2
   seconds ("anything new?" — answer almost always no). The replacement: each client holds a
   persistent WebSocket to the Keepalive system. When any backend service has news (match,
   message), it calls a lightweight Gateway service that wraps the event as a tiny Protocol
   Buffers **Nudge** — effectively "something is new" — pushed down the socket; the client then
   fetches the actual data through normal APIs. Nudges are best-effort by design: a lost nudge
   is healed by the next one or by the client's periodic fallback fetch.
6. **Edge.** A custom API gateway (TAG, built on Spring Cloud Gateway) fronts the public APIs
   (route management, auth filters), per Tinder's engineering blog.

Component list (plain text):
- Mobile clients -> API gateway (TAG)
- Kafka (per-user-partitioned profile/location updates)
- Geosharded Elasticsearch cluster (S2-cell shards; coordinating/master/data nodes)
- Recs service (shard fan-out, filtering, ML ranking; TinVec/vector search)
- Swipes service -> stream (Kinesis) -> match workers -> Likes cache (Redis) -> matches store (DynamoDB)
- S3 (left-swipe/analytics archive)
- Keepalive system: backend services -> Gateway (protobuf Nudge) -> WebSocket -> app
- Push notifications as the out-of-app channel

## Signature ideas
- **Geosharding on S2 cells.** Shard the index by geography, not by user ID hash. Google S2 maps
  the sphere to cells via a space-filling Hilbert curve, so IDs that are numerically close are
  physically close — cheap "which cells cover this circle?" math. Tinder sizes shards by user
  density (small cells in Tokyo, huge in Wyoming) so every shard carries similar load. Result:
  queries touch ~one shard instead of the whole world; the published outcome was roughly 20x
  more computational capacity for the same footprint.
- **Fan-out at the borders.** A user's 100-mile circle can straddle shard boundaries, so queries
  fan out to all overlapping geoshards, and a user moving (or using Passport to teleport to
  another city) migrates between shards. The shard-size trade-off is explicitly tuned: bigger
  shards mean fewer fan-outs but more irrelevant data per query.
- **Consistency in a moving world (part 3 of their series).** Guaranteeing a user appears in
  exactly the right shard despite races: Kafka partitions by user for ordered writes;
  Elasticsearch's near-real-time refresh gap is handled (forcing refresh via the Get API path
  when needed); background reindexing reconciles the search index against the source of truth
  after moves or failures.
- **Time-zone-aware load balancing.** Peak hours follow the sun. By co-locating geoshards from
  different longitudes on the same hardware, one node serves Tokyo's evening while London sleeps
  — flattening the diurnal load curve instead of provisioning every region for its own peak.
- **Match = stream + cache lookup.** Match detection is not a database join: it's a worker
  reading a swipe stream and probing a Likes cache keyed by the pair. The write path (record my
  like) and the match path (did they like me?) meet in O(1), which is how a billion-plus daily
  swipes stay cheap.
- **The Nudge pattern (notify-then-fetch).** Instead of pushing full payloads over WebSockets,
  push a tiny best-effort "something changed" signal and let the client pull through the normal
  API. This keeps the realtime tier stateless and forgiving — losing a nudge costs a moment, not
  a message — and it replaced 2-second polling from every open app.

## Key numbers
- ~1.6 billion swipes per day — Tinder press stat, repeated in "How Tinder Scaled to 1.6 Billion
  Swipes per Day" (systemdesign.one, 2024) — https://newsletter.systemdesign.one/p/tinder-architecture
- ~26 million matches per day (same 2024 writeup, from Tinder-published figures) —
  https://newsletter.systemdesign.one/p/tinder-architecture
- ~75 million monthly users; ~20x computational-capacity gain from geosharding; S2 level 7 ≈ 45
  mi / level 8 ≈ 22.5 mi cells (ByteByteGo summary, 2024, of Tinder's 2019 series) —
  https://blog.bytebytego.com/p/how-tinder-recommends-to-75-million
- Recommendations limited to a 100-mile max distance (Tinder geosharding series, 2019) —
  https://medium.com/tinder-engineering/geosharded-recommendations-part-1-sharding-approach-d5d54e0ec77a
- Pre-2019 clients polled every 2 seconds for updates (Tinder Keepalive post, 2019) —
  https://medium.com/tinder-engineering/how-tinder-delivers-your-matches-and-messages-at-scale-504049f22ce0
- ES8 migration: p99 search latency down 12–56%, +6.5% match rate from deep-learning models,
  >90% of recommendations served by a single Elasticsearch cluster, shards per node 3 → 14
  (Life at Tinder blog, mid-2020s; planning began 2021 on end-of-life ES6) —
  https://www.lifeattinder.com/blog/tinders-migration-to-elasticsearch-8
- ELO-style score officially retired in favor of a dynamic engagement model (Tinder press, 2019)
  — https://www.tinderpressroom.com/powering-tinder-r-the-method-behind-our-matching

## Evolution timeline
- **~2012–2017** — Startup stack on AWS; recommendations from a single shared Elasticsearch
  index; clients poll for updates; ELO-style desirability ranking; TinVec embeddings presented
  at MLconf 2017.
- **2019** — The two big published re-architectures land: the three-part geosharded
  recommendations series (S2-based shards, consistency machinery) and the Keepalive/Nudge
  WebSocket system replacing 2-second polling. ELO publicly retired.
- **~2020–2022** — API gateway (TAG) built on Spring Cloud Gateway; ElastiCache auto-discovery
  taming; streaming swipe pipeline matures.
- **2021–~2024** — Elasticsearch 6 (EOL) → 8 migration: zero-outage cutover, vector/kNN search
  unlocked, deep-learning ranking, geosharding retained after re-evaluation (12% vs 50% CPU),
  denser shard packing.

## Visualization hooks
1. **The world, cut to fit its people.** A map where S2 cells subdivide until each holds equal
   users: tiny tiles over NYC/Tokyo, giant ones over deserts — the single most Tinder-specific
   image there is.
2. **The Hilbert curve thread.** One continuous curve snaking across a map, showing why nearby
   places get nearby cell IDs — then bend the curve into a straight line to show shards as
   contiguous segments.
3. **Border fan-out.** A user's 100-mile circle overlapping three shards; three query arrows out,
   one merged, ranked card stack back.
4. **Swipe rivers.** Cards swiped left flow into a gray archive pipe (S3); right-swipes flow into
   a bright stream where a worker holds each one up against the Likes cache — when two arrows
   meet, a match spark. Show the asymmetry: most volume is left.
5. **Polling vs. Nudge.** Split screen: thousands of phones shouting "anything new?" every 2
   seconds vs. silence until one tiny nudge arrow drops down a WebSocket and the phone reaches
   up to fetch. A load graph collapsing underneath.
6. **Follow one match end-to-end.** Alice in shard A likes Bob; Bob's earlier like sits in the
   cache; worker matches them; two nudges race down two sockets; both screens bloom "It's a
   Match!" — a single timeline with every hop labeled.
7. **The sun-following load curve.** A rotating globe with shard load histograms; co-hosted
   Tokyo+London shards on one server showing the flattened 24h curve.
8. **A user takes a trip.** Passport/travel: a profile document visibly lifted out of the LA
   shard and re-indexed into the Paris shard, with the Kafka ordering rail keeping updates in
   sequence.

## Sources
- "Geosharded Recommendations Part 1: Sharding Approach" — Frank Ren, Tinder Tech Blog (Medium),
  2019. https://medium.com/tinder-engineering/geosharded-recommendations-part-1-sharding-approach-d5d54e0ec77a
  — Why one index failed; S2 choice; shard sizing. Primary. (Medium blocks some automated
  fetchers; content verified via secondary summaries.)
- "Geosharded Recommendations Part 2: Architecture" — Xiaohu Li, Tinder Tech Blog, 2019.
  https://medium.com/tinder-engineering/geosharded-recommendations-part-2-architecture-3396a8a7efb
  — Index-per-shard layout, routing, replica scaling. Primary.
- "Geosharded Recommendations Part 3: Consistency" — Devin Thomson, Tinder Tech Blog, 2019.
  https://medium.com/tinder-engineering/geosharded-recommendations-part-3-consistency-2d2cb2f0594b
  — Kafka ordering, ES refresh semantics, reindexing. Primary.
- "How Tinder delivers your matches and messages at scale" — Dimitar Dyankov, Tinder Tech Blog,
  2019. https://medium.com/tinder-engineering/how-tinder-delivers-your-matches-and-messages-at-scale-504049f22ce0
  — Keepalive, Nudges, protobuf gateway, end of 2-second polling. Primary.
- "Tinder's migration to Elasticsearch 8" — Life at Tinder blog, mid-2020s.
  https://www.lifeattinder.com/blog/tinders-migration-to-elasticsearch-8 — ES6→ES8, latency and
  match-rate gains, geosharding re-validated. Primary.
- "Powering Tinder — The Method Behind Our Matching" — Tinder Newsroom, 2019.
  https://www.tinderpressroom.com/powering-tinder-r-the-method-behind-our-matching — ELO retired;
  what the ranking considers. Primary (PR, low technical depth).
- "Personalized User Recommendations at Tinder (TinVec)" — MLconf SF session listing, 2017.
  https://mlconf.com/sessions/personalized-user-recommendations-at-tinder-the-t/ — Swipe-based
  user embeddings. Primary (talk abstract).
- "How Tinder Recommends To 75 Million Users with Geosharding" — ByteByteGo newsletter, 2024.
  https://blog.bytebytego.com/p/how-tinder-recommends-to-75-million — Faithful synthesis of the
  3-part series with cell sizes and the 20x figure. Secondary.
- "How Tinder Scaled to 1.6 Billion Swipes per Day" — systemdesign.one newsletter, 2024.
  https://newsletter.systemdesign.one/p/tinder-architecture — Swipe pipeline, Likes cache,
  DynamoDB/Kinesis details, aggregated scale stats; cites Tinder posts and AWS talks. Secondary.
- "Designing Tinder" — HighScalability, 2022. https://highscalability.com/designing-tinder/ —
  Design-oriented overview (geosharded ES, streams, WebSockets). Secondary (interview-style
  reconstruction, not Tinder-authored).
