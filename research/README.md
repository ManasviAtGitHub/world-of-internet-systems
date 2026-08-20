# World of Internet Systems — Research Base

Source material for a visualization content series on the system designs behind the internet's
most popular services. **Research only — no visual/design decisions have been made yet.**

Every file follows the same template so content can be compared and mined uniformly:

1. **One-line hook** — the single most compelling concept the system teaches
2. **The core problem** — what hard problem, at what scale
3. **Architecture overview** — components + the end-to-end data flow
4. **Signature ideas** — the 3–6 famous techniques
5. **Key numbers** — every stat carries the year it was stated + source URL
6. **Evolution timeline** — major architecture shifts
7. **Visualization hooks** — concrete ideas for what to *draw* (the section to mine when designing)
8. **Sources** — annotated URLs, each marked primary / secondary / paper / RFC

Gathered 2026-08-20 by parallel research agents from primary sources (official engineering
blogs, original papers, conference talk writeups, postmortems).

## Tier 1 — Flagships (`tier1-flagships/`)

| File | Teaches |
|---|---|
| `netflix.md` | Open Connect CDN inside ISPs, chaos engineering, press-Play-to-frames flow |
| `youtube.md` | Upload→transcode pipeline, Argos custom transcoding silicon, Vitess origins |
| `whatsapp.md` | 2M connections/server on Erlang, ~50 engineers for 900M users, E2E message journey |
| `twitter.md` | Push-vs-pull timelines, the celebrity problem, Snowflake IDs, the open-sourced ranker |
| `instagram.md` | Sharded Postgres + the famous ID bit-layout, 3-engineers-14M-users era |
| `uber.md` | H3 hexagonal geo-index, dispatch/matching, monolith→DOMA evolution |
| `google-search.md` | Crawl→index→serve, PageRank intuition, the GFS→MapReduce→Bigtable→Spanner lineage |
| `amazon.md` | The 2002 API mandate, the Dynamo paper (consistent hashing, quorums), Prime Day scale |

## Tier 2 — Well documented (`tier2-well-documented/`)

| File | Teaches |
|---|---|
| `discord.md` | Elixir guild processes, Cassandra→ScyllaDB trillion-message migration, voice SFU |
| `slack.md` | Flannel edge cache, 500ms worldwide fan-out, workspace sharding → Vitess |
| `spotify.md` | P2P origins (and why dropped), 8M events/sec delivery, Discover Weekly pipeline |
| `zoom.md` | SFU multimedia routing, the 2020 10M→300M scale-up, colo+cloud burst |
| `dropbox.md` | Block-level sync, the move OFF AWS (Magic Pocket), Rust sync engine rewrite |
| `airbnb.md` | Search/ranking, monolith→SOA migration, payments across currencies |
| `stripe.md` | Idempotency keys, API versioning, the life of a card charge |
| `shopify.md` | Pod architecture, the modular-monolith stance, live BFCM flash-sale stats |
| `ticketmaster-booking.md` | Seat locking/holds, virtual waiting rooms, the 2022 Eras onslaught |
| `reddit.md` | Comment trees + precomputed sorts, the "hot" ranking math, r/place |
| `pinterest.md` | The classic MySQL sharding post + ID scheme, visual search, HBase home feed |
| `linkedin.md` | Why Kafka was invented (the log-centric idea), Espresso, FollowFeed |
| `tinder.md` | Geosharded recommendations (Elasticsearch), swipe→match pipeline |
| `wikipedia.md` | Billions of pageviews on aggressive caching and a nonprofit budget — fully public infra |

## Tier 3 — Internet plumbing (`tier3-internet-plumbing/`)

| File | Teaches |
|---|---|
| `url-to-page.md` | Keyboard→GPU with per-stage latency budget (DNS, TCP, TLS 1.3, QUIC, render) |
| `dns.md` | Recursive resolution tree, anycast roots (2,004 instances, live-verified), DoH |
| `bgp.md` | Path-vector trust model, RPKI, the Facebook Oct 2021 outage inside+outside |
| `cdn-cloudflare.md` | Anycast, tiered caching, absorbing a 31.4 Tbps DDoS, edge isolates |
| `load-balancing.md` | L4 vs L7, power-of-two-choices, consistent hashing, Maglev |
| `queues-kafka.md` | The log abstraction (Kreps), partitions/offsets/ISR, why queues decouple |
| `databases-at-scale.md` | Replication, sharding, CAP/PACELC, Raft intuition, the standard scaling ladder |

## ⚠️ Verify before publishing

Claims the research flagged as weaker-sourced — re-check these before they appear in any visual:

- **WhatsApp** ~100B messages/day — traces only to a Q3 2020 earnings call (via TechCrunch), not an engineering source.
- **YouTube** ~2.5B MAU — analyst estimate; Google's last official figure was 2B+ (2019).
- **Cloudflare** "95% of internet population within 50ms" and "~23% of websites" — aggregator-sourced; check cloudflare.com/network.
- **Pinterest** Pinnability "~30% engagement lift" — from the 2015 post; re-check.
- **Zoom** is the thinnest system overall: media-plane internals rest on one 2019 blog post, a 2019 whitepaper, and trade press.
- **Ticketmaster** file mixes disclosed facts with standard reservation-system patterns — each is marked which in-file.
- **Airbnb** calendar-consistency content is labeled "inferred design", not Airbnb-documented.
- **Reddit** vote fuzzing is deliberately under-documented by Reddit (anti-abuse); mechanics stay vague.
- Several primary blogs (netflixtechblog on Medium, blog.x.com, cloudflare.com/learning) 403-block fetchers — facts from them were cross-verified via mirrors, talk writeups, and search excerpts; the primary URLs are still cited.

## Not covered (deliberately)

TikTok's recommender internals, Google ranking specifics, HFT systems — genuinely secret;
content would be speculation.

## Next step

Design phase: decide the visual language and format for the series, mining the
**Visualization hooks** section of each file. Nothing visual has been built yet.
