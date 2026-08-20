# Instagram

## One-line hook
Instagram is proof that boring technology scales: three engineers rode plain Django, PostgreSQL, and Redis to 14 million users — and their one clever invention, a 64-bit ID that encodes time and shard inside itself, became an industry pattern.

## The core problem
Store and serve an exploding firehose of photos and social actions (at sharding time in 2011: 25+ photos and 90+ likes per second) with a tiny team, on rented cloud machines, without inventing new infrastructure. The specific hard sub-problem: once one PostgreSQL box can't hold the data, how do you split it across many machines while keeping IDs unique, time-sortable (feeds are reverse-chronological), and small enough for a 64-bit integer — without running an extra ID service?

## Architecture overview
End-to-end data flow (early era, ~2011-2012, per the Instagram engineering blog and HighScalability):

1. **Upload**: Phone posts a photo → nginx → Gunicorn/Django app servers (stateless, easily scaled from a few to 25+ EC2 instances) → image bytes go to Amazon S3 (served via CloudFront CDN); metadata (user, caption, location, tags) is written to PostgreSQL.
2. **ID + shard routing**: A PL/pgSQL function inside the target shard mints the photo's 64-bit ID: 41 bits of millisecond timestamp (custom epoch) | 13 bits of logical shard ID | 10 bits of per-shard sequence (mod 1024) — so ~1,024 IDs per shard per millisecond, IDs sort by time, and any ID tells you which shard owns it. Thousands of logical shards are implemented as PostgreSQL schemas mapped onto far fewer physical servers, so rebalancing is moving schemas, not resharding data.
3. **Fan-out to feeds**: An async task (Gearman queue, ~200 Python workers) pushes the new photo ID into each follower's feed list in Redis — fan-out-on-write, same family as Twitter. Redis also held the activity feed, sessions, and a ~300M-entry photo-ID→user-ID map for shard lookup. As feed/inbox data outgrew RAM economics, these moved to Cassandra (from 2012).
4. **Read**: Opening the app reads the follower's precomputed feed list, hydrates photo metadata from Postgres/memcached, and pulls images from the CDN. Until 2016 the feed was strictly reverse-chronological; after 2016 a machine-learned ranking reorders the candidate posts per user.
5. **Post-acquisition (2012-2014+)**: The whole stack migrated from AWS into Facebook data centers ("Instagration", ~20B photos moved with no downtime); the social graph moved onto Facebook's TAO graph store, and Instagram adopted Facebook infra (Everstore for blobs, Scuba, Tupperware) while keeping Django at the web tier.

Component list (plain text):
- nginx → Gunicorn → Django (Python) app tier
- PostgreSQL, sharded via logical schemas; pgbouncer connection pooling; streaming replication
- Custom in-database 64-bit ID generator (time | shard | sequence)
- Redis (feeds, activity, sessions, ID maps) → Cassandra (feed, Direct inbox, fraud) 
- memcached cluster
- Gearman async task queue (later Celery/RabbitMQ)
- S3 + CloudFront (photos) → Facebook Everstore after migration
- Solr for geo-search
- TAO (social graph) after Facebook infra migration
- ML feed-ranking pipeline (post-2016)

## Signature ideas
- **The Instagram ID scheme (the famous bit layout).** 64 bits = 41-bit millisecond timestamp + 13-bit logical shard ID + 10-bit sequence. Generated inside PostgreSQL itself with a PL/pgSQL function — no separate ID service like Twitter's Snowflake, no coordination, and `ORDER BY id` equals `ORDER BY created_at`. One of the most-copied blog posts in system design.
- **Logical shards over physical shards.** Thousands of Postgres schemas act as logical shards mapped to a handful of physical machines; capacity growth means moving whole schemas to new hardware rather than rewriting rows. This decoupling of "how data is split" from "where it lives" became standard sharding advice (Pinterest later did a close cousin with MySQL).
- **"3 engineers, 14 million users."** Instagram's early philosophy — keep it simple, don't reinvent the wheel, use proven boring technology (Django, Postgres, Redis, S3) — let 3 engineers run a 14M-user, terabytes-of-photos service in late 2011. The counter-example to resume-driven architecture.
- **Fan-out-on-write feeds in RAM.** Precomputing follower feed lists in Redis (later Cassandra) at post time made reads cheap; Instagram repeatedly published the memory tricks this required (e.g., using Redis hashes instead of plain keys to cut a 300M-key mapping from ~21GB to ~5GB).
- **Chronological → ranked feed (2016).** With growth, users missed ~70% of feed posts, so Instagram introduced ML ranking predicting a handful of engagement outcomes (time spent, like, comment, save, profile tap) — the canonical case study of a product outgrowing reverse-chronological order.
- **Cassandra at Instagram scale, then Rocksandra.** Cassandra (2012→) replaced Redis for feed and Direct inbox; by 2018 Instagram ran one of the largest Cassandra fleets and open-sourced "Rocksandra" — swapping Cassandra's Java storage engine for C++ RocksDB, cutting GC stalls from ~2.5% to ~0.3% and P99 reads to ~20ms (~10x tail-latency win).

## Key numbers
- 14M users, 150M photos, terabytes of storage, 100+ EC2 instances, 3 engineers — 2011, HighScalability (from Instagram's own "What Powers Instagram" post): https://highscalability.com/instagram-architecture-14-million-users-terabytes-of-photos/ and https://instagram-engineering.com/what-powers-instagram-hundreds-of-instances-dozens-of-technologies-adf2e22da2ad
- 25+ photos/sec and 90+ likes/sec at the time of sharding; ID = 41+13+10 bits; 1,024 IDs/shard/ms; several thousand logical shards — 2011, "Sharding & IDs at Instagram": https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
- ~1.3MB memory per raw PostgreSQL connection (why pgbouncer), 12 Postgres replicas, 25+ Django instances, ~200 async workers — 2011-2012, Instagram engineering / HighScalability: https://highscalability.com/instagram-architecture-14-million-users-terabytes-of-photos/
- 30M users and 13 employees at the April 2012 Facebook acquisition ($1B) — 2012, widely reported; see Wired's migration story context: https://www.wired.com/2014/06/facebook-instagram/
- ~20 billion photos and ~200M active users moved from AWS to Facebook data centers with ~8 engineers, no user-visible downtime — 2014, Data Center Knowledge / Wired coverage of the "Instagration": https://www.datacenterknowledge.com/archives/2014/06/27/instagram-migrates-from-amazons-cloud-into-facebook-data-centers
- Users were missing ~70% of feed posts before ranked feed; after ranking, ~90% of friends' posts seen (800M+ users) — 2016-2018, Instagram via TechCrunch "How Instagram's algorithm works": https://techcrunch.com/2018/06/01/how-instagram-feed-works/
- Rocksandra: GC stall 2.5%→0.3%, ~10x tail latency reduction, P99 ~20ms — 2018, "Open-sourcing a 10x reduction in Apache Cassandra tail latency": https://instagram-engineering.com/open-sourcing-a-10x-reduction-in-apache-cassandra-tail-latency-d64f86b43589

## Evolution timeline
- **2010 (Oct)**: Launch — Django + single Postgres on AWS; 25K users on day one.
- **2011**: Growth to 14M users with 3 engineers; sharded PostgreSQL with the custom ID scheme (Sept 2011 blog post); Redis feeds; "What Powers Instagram" post (Dec 2011).
- **2012**: Facebook acquires Instagram (30M users, 13 employees); Cassandra adoption begins (feed, Direct, fraud); Android launch doubles load overnight (1M new users in 12 hours).
- **2013-2014**: "Instagration" — migration off AWS into Facebook data centers; adoption of Facebook infra (TAO for the social graph, Everstore for photos, internal tooling).
- **2016**: Ranked feed replaces pure chronological; ML pipeline predicting engagement outcomes.
- **2017-2018**: "Scaling Instagram Infrastructure" QCon talks (multi-region, Django still at the front); Rocksandra open-sourced (2018); public explanation of feed ranking signals (2018).
- **2019+**: Explore recommender detailed in "Powered by AI" post (two-stage candidate retrieval + ranking, account embeddings).

## Visualization hooks
- The 64-bit ID as a luggage tag: zoom into one integer splitting into [41 bits time | 13 bits shard | 10 bits sequence], with arrows showing "sort by time" and "route to shard 1341" falling straight out of the number.
- Logical vs physical shards: thousands of small schema boxes sliding between a few big database machines as load grows — rebalancing as moving boxes, not repacking them.
- "3 engineers" scale panel: three stick figures vs a wall of 100+ server racks and 14M user icons.
- Fan-out-on-write: one photo upload splitting into follower feed lists in Redis, with the Justin Bieber-style hot account annotated as the stress case.
- Before/after 2016 feed: the same set of posts as a time-ordered river vs a reranked stack, with "70% missed" shrinking to "90% seen."
- The Instagration: a moving-truck diagram — 20B photos driving from AWS-land to Facebook data centers over a Direct Connect bridge, "engine swapped while driving."
- Rocksandra: Cassandra with its Java engine pulled out and a C++ RocksDB engine slotted in; a tail-latency histogram with the P99 tail collapsing 10x.

## Sources
- "Sharding & IDs at Instagram" — Instagram Engineering (Mike Krieger et al.), 2011. The famous ID bit-layout and schema-based logical sharding post. Primary. https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
- "What Powers Instagram: Hundreds of Instances, Dozens of Technologies" — Instagram Engineering, 2011. Full early stack tour: Django, Postgres, Redis, Gearman, S3, monitoring. Primary. https://instagram-engineering.com/what-powers-instagram-hundreds-of-instances-dozens-of-technologies-adf2e22da2ad
- "Instagram Architecture: 14 Million Users, Terabytes of Photos, 100s of Instances, Dozens of Technologies" — HighScalability, 2011. Condensed numbers + principles from Instagram's own posts. Secondary. https://highscalability.com/instagram-architecture-14-million-users-terabytes-of-photos/
- "Storing hundreds of millions of simple key-value pairs in Redis" — Instagram Engineering, 2011. The Redis-hash memory optimization (21GB→5GB) behind feed/ID maps. Primary. https://instagram-engineering.com/storing-hundreds-of-millions-of-simple-key-value-pairs-in-redis-1091ae80f74c
- "Open-sourcing a 10x reduction in Apache Cassandra tail latency" — Instagram Engineering, 2018. Cassandra since 2012 (Feed, Direct, fraud), Rocksandra design and results. Primary. https://instagram-engineering.com/open-sourcing-a-10x-reduction-in-apache-cassandra-tail-latency-d64f86b43589
- "Instagram Migrates from Amazon's Cloud into Facebook Data Centers" — Data Center Knowledge, 2014. The Instagration: 20B photos, no downtime, VPC + Direct Connect route. Secondary (reporting Instagram/Wired statements). https://www.datacenterknowledge.com/archives/2014/06/27/instagram-migrates-from-amazons-cloud-into-facebook-data-centers
- "TAO: The power of the graph" — Facebook Engineering, 2013. The graph store Instagram's social graph landed on post-migration; read-optimized, cache-first, MySQL-backed. Primary (for TAO itself). https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/
- "How Instagram's algorithm works" — TechCrunch, 2018. Instagram's own press briefing on feed ranking signals and the 70%/90% numbers. Secondary (official briefing coverage). https://techcrunch.com/2018/06/01/how-instagram-feed-works/
- "Powered by AI: Instagram's Explore recommender system" — Instagram/Facebook AI blog, 2019. Two-stage retrieval+ranking for Explore, account embeddings ("ig2vec"). Primary. https://ai.meta.com/blog/powered-by-ai-instagrams-explore-recommender-system/
- "Scaling Instagram Infrastructure" — Lisa Guo, QCon London 2017 (InfoQ). Multi-datacenter Django deployment, scale-out/scale-up/team-scale framing. Primary (talk). https://www.infoq.com/presentations/instagram-scale-infrastructure/
