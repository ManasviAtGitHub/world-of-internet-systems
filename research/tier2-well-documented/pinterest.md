# Pinterest

## One-line hook
Pinterest is the "boring sharding done perfectly" story: after flirting with every NoSQL fad during hypergrowth, they bet on manually sharded MySQL with self-describing 64-bit IDs — and separately built one of the first production visual search engines and an in-RAM random-walk recommender over a 3-billion-node graph.

## The core problem
Survive doubling traffic every six weeks (0 → tens of billions of page views/month in two years) with a handful of engineers, then serve a home feed that is not a timeline but a ranked shopping-catalog of images, where the corpus is billions of pins repinned across billions of boards. Two distinct hard problems emerged: (1) partition relational data so it scales linearly without distributed-database magic; (2) recommend visually — because pins are images, not text, discovery needs image understanding and graph walks, not keyword search.

## Architecture overview
End-to-end data flow (per the 2015 sharding post, the 2013 HighScalability talk writeup, and the 2014-2015 feed posts):

1. **Write path (saving a pin)**: App/web client → load balancer → stateless web/API tier (Python/Django originally) → the pin is written to a MySQL shard. The shard is chosen up front (ideally co-located with the board it belongs to), MySQL's auto-increment supplies a **local ID**, and the public **64-bit ID** is assembled as `(shard_id << 46) | (type_id << 36) | local_id` — 16 bits shard, 10 bits type (pin, board, user, ...), 36 bits local row, 2 bits reserved. Any server can decode any ID to find its shard with zero lookups.
2. **Data model on the shards**: Two kinds of tables — **object tables** (pins, boards, users: local_id → JSON blob) and **mapping tables** (relationships like board→pins, user→followers: composite keys with an ordering column). No joins, no foreign keys, no cross-shard transactions; anything relational is done by ID lookups through a thick memcached layer. Shard topology lives in config (moved carefully, never rebalanced automatically).
3. **Feed generation (post-2014 "smart feed")**: When someone you follow saves a pin, a **SmartFeed worker** (driven by the PinLater async job system) scores the pin per follower with the **Pinnability** ML models and writes (user, score, pin) rows into an HBase-backed **pool** of available-but-unseen content. Zen, Pinterest's graph service, abstracts HBase underneath.
4. **Read path (opening the app)**: The **SmartFeed service** concurrently pulls (a) top-scored unseen pins from the pools cluster and (b) already-materialized feed content from a read-optimized **materialized content** HBase cluster, then splices in recommendations and ads — best-first ordering, not chronological. A hot-standby HBase cluster in another availability zone (sync lag of a few hundred ms) answers speculative retries past the p99.9 latency, giving better-than-four-nines availability.
5. **Discovery path**: Related Pins/home feed recommendations come from **Pixie** — random walks from your recent pins over an in-memory bipartite pin↔board graph (3B nodes, 17B edges pruned to ~120GB of RAM per server). Visual search runs a deep CNN embedding over every image so a crop of a lamp in one photo retrieves visually similar buyable pins.

Component list (plain text):
- Stateless web + API tier (Django-era Python), ELB/HAProxy
- Sharded MySQL fleet (object tables + mapping tables), 64-bit self-routing IDs
- memcached + Redis (Redis notably held the follower graph early on)
- PinLater asynchronous job execution system
- Zen graph storage service (on HBase)
- SmartFeed worker / SmartFeed content generator / SmartFeed service
- HBase clusters: pools (write-heavy), materialized feed (read-heavy), hot standby
- Pinnability ML scoring models
- Pixie random-walk recommender (graph in RAM)
- Visual search stack: CNN embeddings, ANN index, Lens camera search

## Signature ideas
- **The self-routing 64-bit ID.** Packing shard ID + type + local ID into one integer (16/10/36 bits) means the ID *is* the routing table: given any object ID you can compute its database, and 65K logical shards can migrate between physical machines without changing a single ID. Capacity by construction: ~68 billion objects per shard type per shard.
- **Logical shards, boring MySQL.** Pinterest ran thousands of logical shards (databases) across few physical machines; growth = moving whole databases to new hardware. Their loudest lesson from the 2012 near-death experience: they removed Cassandra, MongoDB, and membase mid-crisis and standardized on MySQL + memcached + Redis + Solr — "mature, well-known, well-liked" tech that fails in understood ways.
- **The feed as a scored pool, not a timeline.** The 2014 smart feed re-imagined the home feed: separate "available" (unseen, scored) from "seen" (materialized) content, store both in HBase keyed by (user, score, pin) so a scan returns best-first, and accept eventual delivery — a pin may arrive in your feed minutes later if that makes it better ranked. This is the cleanest public description of a feed built around ranking pools rather than fanout lists.
- **Pinnability.** Pinterest's umbrella name (2015) for the ML models predicting the probability you'll engage with a pin, applied at feed-write time by SmartFeed workers; the blog reported roughly a 30% lift in home feed engagement after launch. An early, well-documented example of "ranking everything" before it was universal.
- **Pixie: random walks instead of matrix factorization.** Real-time recommendations by running thousands of short random walks from query pins across the pin↔board graph held entirely in RAM on each server — 60ms p99, ~1,200 req/s per machine, no precomputed recommendations at all. Biased walks + graph pruning beat the old Hadoop batch pipeline by ~50% on engagement.
- **Visual search as a first-class retrieval system.** Pinterest published one of the first production visual-search architectures (KDD 2015): incrementally fingerprint billions of images, extract deep CNN features, serve nearest-neighbor lookups cheaply enough for a startup; then Lens (2017) pointed the camera at the real world, and a unified 512-d embedding (KDD 2019) replaced three specialized embeddings to serve 600M+ visual searches/month.

## Key numbers
- 0 → tens of billions of page views/month in two years; stack at Jan 2013: 180 web engines, 240 API engines, 88 sharded MySQL DBs (cc2.8xlarge) each with a slave, 110 Redis instances, 200 memcached instances — 2013, HighScalability "Scaling Pinterest": https://highscalability.com/scaling-pinterest-from-0-to-10s-of-billions-of-page-views-a/
- ID layout 16-bit shard | 10-bit type | 36-bit local (2 bits reserved) → 65K shards, ~68B objects per shard; sharding system built ~2012, post published 2015 — Pinterest Engineering, "Sharding Pinterest: How we scaled our MySQL fleet": https://medium.com/pinterest-engineering/sharding-pinterest-how-we-scaled-our-mysql-fleet-3f341e96ca6f
- Home feed HBase system: hundreds of terabytes, millions of operations/second, hundreds of millions of calls/day; standby cluster sync within a few hundred ms; better than 99.99% availability — 2014, "Building a scalable and available home feed" (Pinterest engineering, mirrored on pingineering.tumblr.com): https://medium.com/pinterest-engineering/building-a-scalable-and-available-home-feed-6a343766bb6
- Smart feed launched with best-first ordering and available/seen pool separation — 2014, "Building a smarter home feed": https://medium.com/pinterest-engineering/building-a-smarter-home-feed-ad1918fdfbe3
- Pinnability ML scoring reported ~30% home feed engagement lift — 2015, "Pinnability: Machine learning in the home feed": https://medium.com/pinterest-engineering/pinnability-machine-learning-in-the-home-feed-64be2074bf60
- Pixie: 3B nodes / 17B edges pruned into ~120GB RAM (AWS r3.8xlarge, 244GB), <60ms p99, ~1,200 req/s per server, ~100K req/s cluster-wide, up to +50% engagement vs Hadoop pipeline — 2018 (WWW '18), Eksombatchai et al.: https://arxiv.org/abs/1711.07601
- Visual search: 600M+ visual searches/month; unified SE-ResNeXt101 512-d embedding — 2019 (KDD '19), "Learning a Unified Embedding for Visual Search at Pinterest": https://arxiv.org/abs/1908.01707
- First visual search paper (distributed CNN feature pipeline, incremental fingerprinting) — 2015 (KDD '15), "Visual Search at Pinterest": https://arxiv.org/abs/1505.07647

## Evolution timeline
- **2010-2011**: Launch; Rackspace → AWS; small Django + MySQL stack; growth doubling every ~6 weeks.
- **2011-2012**: The technology-sprawl crisis — MySQL + MongoDB + Cassandra + membase + Redis simultaneously; cluster tech failures during hypergrowth; the great simplification back to MySQL/memcached/Redis/Solr.
- **2012**: Manual MySQL sharding with the 64-bit ID scheme goes live (documented publicly in 2015).
- **2013**: "Scaling Pinterest" talk (QCon/HighScalability writeup); follower graph in Redis; PinLater job system era.
- **2014**: Zen graph service on HBase; smart feed replaces chronological home feed (pools + materialized clusters).
- **2015**: Pinnability ML ranking in the home feed; "Visual Search at Pinterest" (KDD) ships visually-similar results and the crop tool.
- **2017**: Lens camera search; Pixie random-walk recommender in production (paper at WWW 2018).
- **2019**: Unified visual embedding (KDD); visual search at 600M+ queries/month; later years: GPU-era rankers and a move of workloads toward Kubernetes and modern ML infra.

## Visualization hooks
- The ID as a map: a 64-bit strip that unfolds into an actual route — [shard 16 | type 10 | local 36] with an arrow flying from the number straight to the right database box among 65,536.
- "The great simplification": a chaotic 2012 rack with MySQL+Mongo+Cassandra+membase logos, arrows crossing; then the 2013 rack with four boring boxes and clean lines — annotate "doubling every 6 weeks during this."
- Smart feed pools: two tanks per user — "available (scored, unseen)" and "seen (materialized)" — with the SmartFeed service ladling best-first from both when the app opens; contrast against a Twitter-style conveyor-belt timeline.
- HBase key trick: rows sorted by (user, score, pin) so "read the top of the table" = "read the best content" — draw the scan head skimming the top rows.
- Pixie: a pinboard galaxy — random-walk trails sparking out from 3 query pins across a pin↔board bipartite graph held inside one machine's RAM outline; counters tally node visits.
- Visual search: a photo of a living room with a crop-box on a lamp; the crop → CNN → embedding point in a cloud of billions → nearest neighbors returned as buyable pins.
- The speculative-read failover: a request racing two HBase clusters (primary vs warm standby in another AZ), whichever answers first wins.

## Sources
- "Sharding Pinterest: How we scaled our MySQL fleet" — Pinterest Engineering Blog (Medium), 2015. The classic: ID bit layout, object/mapping tables, no-joins discipline, shard migration playbook. Primary. https://medium.com/pinterest-engineering/sharding-pinterest-how-we-scaled-our-mysql-fleet-3f341e96ca6f
- "Scaling Pinterest - From 0 to 10s of Billions of Page Views a Month in Two Years" — HighScalability, 2013 (writeup of Yashwanth Nelapati & Marty Weiner's talk). Growth numbers, the NoSQL-purge story, stack inventory, sharding rationale. Secondary (faithful to primary talk; talk slides also on InfoQ/SlideShare). https://highscalability.com/scaling-pinterest-from-0-to-10s-of-billions-of-page-views-a/
- "Building a smarter home feed" — Pinterest Engineering Blog, 2014. Chronological → best-first, available/seen separation, HBase (user, score, pin) key design. Primary. https://medium.com/pinterest-engineering/building-a-smarter-home-feed-ad1918fdfbe3
- "Building a scalable and available home feed" — Pinterest Engineering Blog, 2014 (mirror: https://pingineering.tumblr.com/post/105293275179/building-a-scalable-and-available-home-feed). Pools vs materialized HBase clusters, Zen, PinLater, standby-cluster speculative reads, scale/availability numbers. Primary. https://medium.com/pinterest-engineering/building-a-scalable-and-available-home-feed-6a343766bb6
- "Pinnability: Machine learning in the home feed" — Pinterest Engineering Blog, 2015. The ML scoring layer applied by SmartFeed workers; engagement lift claim. Primary. https://medium.com/pinterest-engineering/pinnability-machine-learning-in-the-home-feed-64be2074bf60
- "Visual Search at Pinterest" — Jing et al., KDD 2015 (arXiv:1505.07647). First production visual-search architecture paper: incremental fingerprinting, CNN features on commodity infra. Primary (peer-reviewed). https://arxiv.org/abs/1505.07647
- "Pixie: A System for Recommending 3+ Billion Items to 200+ Million Users in Real-Time" — Eksombatchai, Leskovec, et al., WWW 2018 (arXiv:1711.07601). The random-walk recommender: graph-in-RAM, biased walks, all serving numbers. Primary (peer-reviewed). https://arxiv.org/abs/1711.07601
- "Learning a Unified Embedding for Visual Search at Pinterest" — Zhai et al., KDD 2019 (arXiv:1908.01707) + companion blog "Unifying visual embeddings for visual search at Pinterest". Multi-task 512-d embedding replacing per-product embeddings; 600M searches/month. Primary. https://arxiv.org/abs/1908.01707
- "Zen: Pinterest's Graph Storage Service" — QCon 2014 talk (Raghavendra Prabhu). The HBase-backed graph model under feeds/following. Primary (talk). https://www.infoq.com/presentations/zen-pinterest-graph-storage-service/
