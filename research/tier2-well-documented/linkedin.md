# LinkedIn

## One-line hook
LinkedIn's feed problem was so painful that the side effect of solving it — Kafka, and the idea that an append-only log should be the backbone of a company's entire data architecture — became more famous than the feed itself.

## The core problem
Integrate one social network's data across dozens of specialized systems. By the late 2000s LinkedIn had a monolith ("Leo"), an Oracle source of truth, a search index, a graph engine, Hadoop, caches, and monitoring — and every pair of systems grew its own fragile point-to-point pipeline (N systems → O(N²) integrations). Meanwhile the feed had the classic activity-stream problem: hundreds of millions of members following people and companies, each feed view needing recent activities from thousands of followed actors, ranked, in ~100ms. LinkedIn's distinctive answers: make every data change an event on a central log (Kafka), and build the feed as fan-out-on-read over per-actor timeline indexes (FollowFeed) instead of Twitter-style fan-out-on-write.

## Architecture overview
End-to-end data flow (feed era, per the 2016 FollowFeed post and "A Brief History of Scaling LinkedIn"):

1. **Write path (a member shares a post)**: Client → frontend services → the post is written to **Espresso**, LinkedIn's source-of-truth distributed document store (partitioned MySQL storage nodes under a document API, Apache Helix managing partition mastership, timeline-consistent replicas). Every change is captured (Databus change-data-capture, later Brooklin) and emitted as events onto **Kafka**.
2. **Indexing**: FollowFeed's index nodes consume the activity events from Kafka and append each activity to the **timeline index** of its actor — a term-partitioned index keyed by actor (member:123, company:456), whose value is a reverse-chronological posting list of that actor's activities, stored in **RocksDB** as a linked list of serialized blobs. Recent/hot entries sit in a Caffeine cache above RocksDB.
3. **Read path (member opens the feed) — fan-out-on-read**: The feed request arrives with the member's followed-actor list (from the graph/follows service) → a **broker** node scatters the query to the index-node partitions owning those actors' timelines → each index node retrieves candidate activities and applies early relevance scoring/filtering locally (pushing ranking down to the data) → the broker gathers, merges, and returns candidates → higher-level feed services (feed-mixer era) blend in jobs, ads, and recommendations, with final ML ranking → response in ~140ms at p99 (2016).
4. **Everything else subscribes to the log**: the same Kafka events feed Hadoop for model training, Samza for stream processing, metrics/monitoring, cache invalidation, and search indexing — the "log-centric" architecture Jay Kreps described in "The Log": every consumer replays the same ordered stream at its own pace.
5. **Historical layers**: before this, feed queries ran on Sensei (a distributed search system); derived/batch data (People You May Know, related profiles) was served from **Voldemort** read-only stores bulk-built in Hadoop and atomically swapped in; services talk **Rest.li** (JSON/HTTP RPC with D2/ZooKeeper-based discovery) — all open-sourced.

Component list (plain text):
- Frontend/API services (Java), Rest.li RPC + D2 dynamic discovery (ZooKeeper)
- Espresso: routers → MySQL-backed storage nodes, Apache Helix cluster management, timeline consistency, Databus/Brooklin change capture
- Kafka: the central event log (all activity data, metrics, DB change streams)
- FollowFeed: Kafka consumers → index nodes (RocksDB timeline indexes + Caffeine cache) + broker scatter-gather; early scoring at the leaves
- Feed relevance tier (feed-mixer + ML rankers)
- Voldemort (Dynamo-style KV + Hadoop-built read-only stores; later replaced by Venice)
- Samza stream processing; Hadoop offline
- Member graph service (the first service ever split from the Leo monolith)
- Galene/Lucene search stack

## Signature ideas
- **Kafka, and why a social network invented it.** LinkedIn's activity data (page views, shares, connects) needed to reach warehouses, search, monitoring, and recommenders; bespoke pipelines between each pair kept breaking and batch ETL was too slow. Kafka (built ~2010, open-sourced 2011, NetDB '11 paper) reframed messaging as a *replicated, partitioned commit log*: producers append, every consumer group reads sequentially at its own offset, retention is time-based not delivery-based. One pipeline, N subscribers, O(N) integrations.
- **The log-centric architecture.** Jay Kreps' 2013 essay "The Log" generalized the lesson: an append-only, ordered log is the unifying abstraction behind databases (WAL), replication, and stream processing — so make the log itself the company-wide integration point and let every system (cache, index, warehouse) be a materialized view over it. Arguably the most influential engineering blog post of its decade; it seeded Kafka Streams, Samza, and the "streaming data platform" industry.
- **FollowFeed: fan-out-on-read done seriously.** Instead of pushing each activity into every follower's inbox (write amplification, celebrity problem), LinkedIn indexes activities once per *actor* and aggregates at read time with scatter-gather brokers. The trick that makes pull viable: partition-local relevance scoring (move computation to the data) plus aggressive caching. The 2016 migration off Sensei cut mobile-feed p99 to ~140ms (5x faster), stored ~20x more history, and halved capex.
- **Espresso: a source-of-truth document store on MySQL.** LinkedIn replaced Oracle-centric storage with an in-house horizontally partitioned document database: MySQL/InnoDB as the storage engine, Apache Helix assigning partition masters/replicas, hierarchical document keys, secondary indexes, and "timeline consistency" (replicas apply the same commit order, slightly behind) — plus a built-in change stream feeding Kafka/Databus. Documented in the SIGMOD 2013 paper and a 2015 engineering post; it carried member profiles and InMail.
- **Voldemort and batch-computed serving.** LinkedIn's Dynamo-inspired key-value store (open-sourced 2009) had a signature mode: read-only stores whose index/data files are built offline in Hadoop, shipped to servers, and atomically swapped — turning "serve yesterday's giant computation" (People You May Know) into a solved deployment problem (FAST '12 paper). Retired in favor of Venice, its purpose-built successor.
- **Rest.li and the service explosion.** Going from the Leo monolith (2003) to 750+ services (2015) required standardized plumbing: Rest.li (open-sourced 2013) gave type-safe REST/JSON RPC with schema'd models, async I/O, and D2 client-side load balancing/discovery via ZooKeeper — LinkedIn's equivalent of Twitter's Finagle.

## Key numbers
- Kafka at LinkedIn: 7+ trillion messages/day across 100+ clusters, ~4,000 brokers, ~100,000 topics, 7 million partitions — 2019, "How LinkedIn customizes Apache Kafka for 7 trillion messages per day": https://engineering.linkedin.com/blog/2019/apache-kafka-trillion-messages
- Kafka handling ~500 billion events/day and LinkedIn at 350M+ members; service count 1 (2003) → 150+ (2010) → 750+ (2015); 2,700 members in launch week 2003 — 2015, "A Brief History of Scaling LinkedIn": https://engineering.linkedin.com/architecture/brief-history-scaling-linkedin
- FollowFeed vs Sensei: mobile feed p99 ~140ms (≈5x faster), ~150ms improvement in p90 page-load latency, ~20x more indexed data, ~50% capex reduction — 2016, "FollowFeed: LinkedIn's Feed Made Faster and Smarter": https://engineering.linkedin.com/blog/2016/03/followfeed--linkedin-s-feed-made-faster-and-smarter
- Kafka open-sourced 2011; original design goals and benchmarks in "Kafka: a Distributed Messaging System for Log Processing" — 2011 (NetDB '11), Kreps/Narkhede/Rao: https://notes.stephenholiday.com/Kafka.pdf (paper mirror; see also https://kafka.apache.org/books-and-papers)
- Espresso design (MySQL storage nodes, Helix, timeline consistency; production since 2012 for use cases like InMail) — 2013 (SIGMOD '13), "On Brewing Fresh Espresso: LinkedIn's Distributed Data Serving Platform": https://dl.acm.org/doi/10.1145/2463676.2465298
- Voldemort read-only stores serving batch-computed data at LinkedIn — 2012 (FAST '12), "Serving Large-scale Batch Computed Data with Project Voldemort": https://www.usenix.org/conference/fast12/serving-large-scale-batch-computed-data-project-voldemort
- "The Log" essay (log-centric architecture manifesto) — 2013, Jay Kreps, LinkedIn Engineering: https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying

## Evolution timeline
- **2003**: Launch; "Leo" — a single monolithic Java web app on one database (Oracle).
- **2003-2006**: First services split out: the in-memory member graph service ("Cloud"), then search; read replicas fed by Databus change capture relieve the primary DB.
- **2008-2010**: Full SOA push; Project Voldemort (Dynamo-style KV, open-sourced 2009); Kafka built to replace fragmented activity-data pipelines (open-sourced January 2011).
- **2011**: "Inversion" — a feature freeze to rebuild tooling/deployment; "Kill Leo" completes the monolith's dismantling; Kafka enters the Apache incubator.
- **2012-2013**: Espresso goes to production as the Oracle-replacing source-of-truth document store (SIGMOD '13); Rest.li + D2 open-sourced; Samza for stream processing; multi-datacenter serving; December 2013: "The Log" essay.
- **2014-2016**: Confluent spins out of LinkedIn to commercialize Kafka (2014); FollowFeed replaces the Sensei-based feed backend (2016) with read-time scatter-gather over RocksDB timeline indexes.
- **2017-2019**: Venice replaces Voldemort for derived-data serving; Brooklin replaces Databus for change streams; Kafka passes 7 trillion messages/day (2019).
- **2020s**: Feed ranking modernization continues (deep/sequential rankers documented in later papers and posts) atop the same log-centric substrate.

## Visualization hooks
- The pipeline-spaghetti before/after: N systems with N² crossing arrows, then the same systems all plugged into one horizontal Kafka log — arrows become tidy taps on a pipe; count the connections shrinking from ~30 to ~10.
- The log itself: an append-only tape with numbered slots; three consumers (search index, Hadoop, cache invalidator) each holding their own bookmark/offset at different positions on the same tape.
- Push vs pull face-off: Twitter's one-tweet-to-millions-of-inboxes explosion next to LinkedIn's opposite — one feed request scattering out to index nodes partitioned by followed actors, results merging back through a broker.
- Anatomy of a timeline index: an actor's ID unlocking a chain of reverse-chronological activity blobs in RocksDB, with a small "early scorer" stamping relevance scores right at the storage node.
- Leo's decomposition as a 12-year exploded diagram: one big block (2003) → 150 blocks (2010) → 750+ blocks (2015), with Rest.li shown as the standard connector between all of them.
- Espresso cutaway: router layer → partitioned MySQL storage nodes with Helix as the conductor reassigning master batons when a node dies; a Databus siphon feeding changes back onto the Kafka log.
- The 7-trillion counter: a day at LinkedIn as a Kafka odometer rolling past 7,000,000,000,000 messages, with 7M partitions as the lanes.
- Voldemort's read-only store swap: Hadoop baking a giant index overnight, shipping it to servers, and an atomic pointer flip switching traffic to the fresh copy.

## Sources
- "The Log: What every software engineer should know about real-time data's unifying abstraction" — Jay Kreps, LinkedIn Engineering, 2013. The log-centric architecture manifesto; why Kafka exists and what it generalizes. Primary. https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
- "A Brief History of Scaling LinkedIn" — Josh Clemm, LinkedIn Engineering, 2015. Leo monolith → member graph service → SOA → Inversion → super blocks/multi-DC; service counts and Kafka event volume. Primary. https://engineering.linkedin.com/architecture/brief-history-scaling-linkedin
- "FollowFeed: LinkedIn's Feed Made Faster and Smarter" — Swapnil Ghike & Shubham Gupta, LinkedIn Engineering, 2016. The definitive fan-out-on-read feed writeup: actor-partitioned timeline indexes, RocksDB blob chains, broker scatter-gather, Sensei migration numbers. Primary. https://engineering.linkedin.com/blog/2016/03/followfeed--linkedin-s-feed-made-faster-and-smarter
- "How LinkedIn customizes Apache Kafka for 7 trillion messages per day" — LinkedIn Engineering, 2019. Scale stats (clusters/brokers/topics/partitions) and LinkedIn's internal Kafka release process. Primary. https://engineering.linkedin.com/blog/2019/apache-kafka-trillion-messages
- "Kafka: a Distributed Messaging System for Log Processing" — Kreps, Narkhede, Rao, NetDB 2011. Original Kafka paper: design goals, pull-based consumers, sequential I/O argument. Primary (peer-reviewed workshop). https://kafka.apache.org/books-and-papers
- "On Brewing Fresh Espresso: LinkedIn's Distributed Data Serving Platform" — SIGMOD 2013. Espresso internals: MySQL storage nodes, Helix, timeline consistency, secondary indexing, change capture. Primary (peer-reviewed). https://dl.acm.org/doi/10.1145/2463676.2465298
- "Introducing Espresso — LinkedIn's hot new distributed document store" — LinkedIn Engineering, 2015. Blog-level Espresso overview and use cases (profiles, InMail). Primary. https://engineering.linkedin.com/espresso/introducing-espresso-linkedins-hot-new-distributed-document-store
- "Serving Large-scale Batch Computed Data with Project Voldemort" — Sumbaly et al., FAST 2012. The read-only-store build-and-swap design behind PYMK-style features. Primary (peer-reviewed). https://www.usenix.org/conference/fast12/serving-large-scale-batch-computed-data-project-voldemort
- "Rest.li: RESTful Service Architecture at Scale" — LinkedIn Engineering, 2013 (open-source announcement; repo: https://github.com/linkedin/rest.li). Typed REST framework + D2 discovery underpinning the 750-service SOA. Primary. https://engineering.linkedin.com/architecture/restli-restful-service-architecture-scale
- "Building the Activity Graph" Parts I-II — LinkedIn Engineering, 2017. Follow-on feed infrastructure detail (activity data modeling above FollowFeed). Primary. https://engineering.linkedin.com/blog/2017/06/building-the-activity-graph--part-i
