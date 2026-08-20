# Message Queues & Kafka

## One-line hook
LinkedIn needed to move log data around, so it rebuilt the humblest data structure in computing — the append-only log — into a system that now carries 7 trillion messages a day and quietly underpins most of the internet's data plumbing.

## The core problem
Two problems, actually. First, **integration sprawl**: with N systems producing data and M systems consuming it, point-to-point pipelines multiply toward N×M bespoke connections, each with its own format, failure modes, and lag. Jay Kreps' "The Log" essay frames the fix: put one durable, ordered log in the middle and the problem collapses to N+M connections — everyone writes to the log, everyone reads from the log, at their own pace. Second, **coupling in time and rate**: if a producer calls a consumer directly, the producer can only go as fast as its slowest consumer, and a consumer crash becomes the producer's outage. A queue decouples them: the producer appends and moves on; the consumer catches up whenever it can; a traffic spike becomes queue depth (visible, survivable backlog) instead of cascading failure.

## How it works
Kafka's mental model in one sentence: a **topic** is a named category of messages, physically split into **partitions**, and each partition is an append-only log file whose entries are numbered by **offset**.

Step by step:
1. A **producer** sends a record (key, value) to a topic. The key is hashed to pick a partition (same key → same partition → per-key ordering). Records are batched and compressed for throughput.
2. The **broker** leading that partition appends the record to its log on disk. Kafka leans on sequential disk I/O, the OS page cache, and zero-copy sendfile — which is why a "dumb" log outruns fancier brokers.
3. **Replication:** each partition has one leader and several followers on other brokers. Followers pull from the leader; those that keep up form the **ISR (in-sync replica set)**. With `acks=all`, the leader confirms a write only after all in-sync replicas have it; `min.insync.replicas` sets the floor. If a leader dies, a controller elects a new leader from the ISR — no committed data lost. This f+1-copies design (vs. 2f+1 majority quorums) is Kafka's chosen durability/cost tradeoff.
4. **Consumers pull.** A consumer reads a partition sequentially from its stored offset and periodically commits that offset back. Messages are NOT deleted when read — the log is retained by time/size policy (or compacted by key), so consumers can rewind and replay history.
5. **Consumer groups:** consumers sharing a group id divide a topic's partitions among themselves — each partition goes to exactly one member (queue semantics within a group, broadcast across groups). When members join or leave, a **rebalance** reassigns partitions.
6. Coordination (broker membership, leader election) historically lived in ZooKeeper; since KIP-500 it's an internal Raft-based quorum (**KRaft**), and Kafka 4.0 dropped ZooKeeper entirely.

Plain-text component list:
- Producer (partitioner, batcher, optional idempotence/transactions)
- Topic → partitions → segments on disk (append-only, offset-indexed)
- Brokers: partition leaders + followers, ISR tracking
- Controller (KRaft quorum) for metadata and leader election
- Consumer groups + group coordinator + committed offsets (stored in an internal topic)

End-to-end flow: a rider's GPS ping at Uber → producer hashes trip-id → partition 42 of topic `locations` → leader on broker 7 appends at offset 918,223 → two ISR followers replicate → ack → the surge-pricing consumer group (reading offset ~918,000) and the fraud-detection group (lagging at ~900,000) each pull it independently; neither knows the other exists.

Contrast with RabbitMQ-style brokers: a classic AMQP broker is a *smart* broker — it routes each message through exchanges to queues, pushes to consumers, tracks per-message acknowledgements, and deletes messages once consumed. Great for task distribution (work disappears when done), weaker for replay, ordering at scale, and fan-out to many independent readers — which is exactly the ground the log abstraction owns. (Tellingly, RabbitMQ later added Kafka-style "Streams" in 3.9, 2021.)

## Signature ideas
- **The log as the unifying abstraction.** Kreps' 2013 essay argues the append-only, totally-ordered log is the common core of database WALs, replication streams, and messaging — so make it a first-class service. State machine replication falls out for free: if two systems consume the same log in the same order, they end up in the same state. This one idea links Kafka to database replication and consensus.
- **Dumb broker, smart consumer.** Kafka's broker doesn't track which messages each consumer has seen — consumers own their offsets. That single design choice (from the 2011 NetDB paper) removes per-message broker state, enables cheap replay/rewind, and lets one written byte serve unlimited readers.
- **Partitions are the unit of everything.** Ordering guarantee, parallelism ceiling, replication unit, and rebalancing token — all the partition. Want more consumer parallelism? You need more partitions. Keys guarantee order only within one partition. Most Kafka design pain (hot keys, rebalance storms, partition-count planning) is partition math.
- **ISR replication: durability without majority quorums.** Kafka tolerates f failures with f+1 replicas (vs 2f+1 for Paxos-style majority writes) by dynamically shrinking the in-sync set and only committing to it. The knobs (`acks`, `min.insync.replicas`, `unclean.leader.election`) form a small, teachable safety dial from "fast and lossy" to "slow and safe" — Netflix famously ran early pipelines at replication-factor 2 with acks=1 and accepted rare loss for latency.
- **Delivery semantics are a spectrum you choose.** At-most-once (fire and forget), at-least-once (retry until acked; duplicates possible — the default posture), and exactly-once within Kafka: idempotent producers (sequence-number dedup per partition) plus transactions (atomic writes across partitions with commit markers), shipped in Kafka 0.11 (2017) after a famously contested "that's impossible" debate.
- **Backpressure by pull.** Because consumers pull and the log retains data for days, a slow consumer just develops *lag* (a measurable offset gap) instead of forcing the producer to slow, buffer, or drop. Consumer lag becomes the single most-watched operational metric — a burndown chart of how far behind reality each system is.

## Key numbers
- Kafka's original benchmark: ~400,000 messages/sec producer throughput, far ahead of ActiveMQ/RabbitMQ in the same test (2011, NetDB paper, LinkedIn) — https://notes.stephenholiday.com/Kafka.pdf (paper PDF mirror; abstract at https://www.microsoft.com/en-us/research/wp-content/uploads/2017/09/Kafka.pdf)
- 2 million writes/sec on three cheap machines — Jay Kreps' benchmark post (2014) — https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines
- LinkedIn passed 1.1 trillion messages/day — Confluent's "four comma club" post (2015) — https://www.confluent.io/blog/apache-kafka-hits-1-1-trillion-messages-per-day-joins-the-4-comma-club/
- 1.4 trillion messages/day over ~1,400 brokers at LinkedIn (2016) — https://www.linkedin.com/blog/engineering/open-source/kafka-ecosystem-at-linkedin
- 7 trillion messages/day, 100+ clusters, 4,000+ brokers, 100,000 topics, 7 million partitions at LinkedIn (2019) — https://www.linkedin.com/blog/engineering/open-source/apache-kafka-trillion-messages
- Netflix Keystone: ~700 billion events/day ingested through 36 Kafka clusters, 4,000+ broker instances (2016) — https://netflixtechblog.com/kafka-inside-keystone-pipeline-dd5aeabaf6bb
- Uber: trillions of messages and multiple petabytes per day through Kafka (2020) — https://www.uber.com/blog/kafka/
- Exactly-once semantics (idempotent producer + transactions) landed in Kafka 0.11 (2017) — https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- Kafka 4.0 removed ZooKeeper; KRaft (Raft-based) metadata quorum is the only mode (2025) — https://kafka.apache.org/blog#apache_kafka_400_release_announcement
- Apache Kafka's own claim: used by more than 80% of the Fortune 100 (current site) — https://kafka.apache.org/

## Who uses it and how
- **LinkedIn (birthplace):** built Kafka in 2010 to unify activity-data and log pipelines; grew from ~1 trillion messages/day (2015) to 7 trillion/day (2019) on 100+ clusters, running a near-upstream fork with scalability patches. The growth curve itself (1T → 1.4T → 7T in four years) is the story.
- **Uber:** one of the largest deployments in the world — trillions of messages and petabytes daily (2020); Kafka feeds rider-driver matching telemetry, fraud detection, and analytics, with cross-region replication for disaster recovery and a "consumer proxy" (uForwarder) layer because tens of thousands of microservices consume it.
- **Netflix (Keystone):** ~700 billion events/day in 2016 via a two-tier design — "fronting" Kafka clusters absorb all producer traffic, separate consumer clusters serve readers — explicitly trading some durability (RF=2, acks=1 early on) for availability and cost, then tightening acks for critical topics after loss incidents.
- **The Confluent spinoff:** Kafka's three creators (Kreps, Narkhede, Rao) left LinkedIn in 2014 to found Confluent — a useful narrative beat: infrastructure so successful it became a company, then a whole "data streaming" market category.
- **Everyone else, via the ecosystem:** banks for payment event streams, retailers for inventory, and per kafka.apache.org, 80%+ of the Fortune 100 — usually not "a message queue" but as the central nervous system pattern Kreps' essay predicted: one log, many materialized views (search indexes, caches, warehouses) all replaying the same stream.

## Visualization hooks
- **N×M → N+M:** an animated tangle of point-to-point pipes between producers and consumers (databases, apps, Hadoop, monitoring) collapsing into a clean hub as the log slides into the middle — the literal diagram from Kreps' essay, begging to be animated.
- **The log as a tape with read heads:** one growing tape (offsets ticking up) with multiple playheads at different positions — real-time consumer near the tip, batch consumer far behind, a new consumer spawning at offset 0 and fast-forwarding through history. Lag is the visible gap.
- **Partitioning as toll lanes:** records with colored keys hashing into lanes; same color always same lane (ordering preserved per key); a hot key piling up one lane while others idle (the hot-partition problem in one frame).
- **Consumer-group rebalance:** 6 partitions, 2 consumers (3 each); a third consumer joins → partitions visibly lift and reassign 2-2-2; one dies → snap back. Show the brief "stop-the-world" freeze that makes rebalances feared.
- **ISR shrink and leader failover:** three replicas ticking along; one follower falls behind and gets ejected from the glowing ISR circle; the leader's rack catches fire; a follower inside the circle is crowned leader; the stale replica is *not* eligible — committed offsets survive.
- **The acks dial:** a physical knob — acks=0 / 1 / all with min.insync.replicas — animated against two meters: latency and "messages lost when a broker dies." Netflix's early RF=2/acks=1 incident as the cautionary vignette.
- **Backpressure, queue vs no queue:** split screen of a traffic spike hitting (a) synchronous RPC chain — requests queue in memory, timeouts cascade, everything reddens; (b) Kafka in the middle — producer unaffected, consumer lag graph swells then burns down overnight.
- **Exactly-once transaction:** a producer writing to two partitions with hollow "pending" markers that solidify simultaneously when the commit marker lands; a consumer in read-committed mode stepping over an aborted batch.

## Sources
- "The Log: What every software engineer should know about real-time data's unifying abstraction" — Jay Kreps, LinkedIn Engineering, 2013. The conceptual spine: logs, state machine replication, data integration, why a log service. (Primary essay by Kafka's co-creator.) https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
- "Kafka: a Distributed Messaging System for Log Processing" — Kreps, Narkhede, Rao, NetDB 2011. Original design rationale (pull consumers, no per-message broker state, sequential I/O) and first benchmarks. (Primary paper.) https://notes.stephenholiday.com/Kafka.pdf
- Apache Kafka official documentation — design section: replication/ISR, delivery semantics, consumer groups, log compaction. (Primary documentation.) https://kafka.apache.org/documentation/#design
- "How LinkedIn customizes Apache Kafka for 7 trillion messages per day" — Jon Lee & Wesley Wu, LinkedIn Engineering, Oct 2019. The 7T/day, 100-cluster, 7M-partition numbers plus fork strategy. (Primary engineering post.) https://www.linkedin.com/blog/engineering/open-source/apache-kafka-trillion-messages
- "Apache Kafka Hits 1.1 Trillion Messages Per Day" — Confluent blog, 2015. Dated milestone for the growth curve. (Primary vendor post, first-party numbers.) https://www.confluent.io/blog/apache-kafka-hits-1-1-trillion-messages-per-day-joins-the-4-comma-club/
- "Benchmarking Apache Kafka: 2 Million Writes Per Second (On Three Cheap Machines)" — Jay Kreps, LinkedIn Engineering, 2014. Reproducible throughput methodology; why logs are fast. (Primary.) https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines
- "Kafka Inside Keystone Pipeline" — Netflix Tech Blog, 2016. 700B events/day, fronting-cluster architecture, durability tradeoffs. (Primary engineering post; site sometimes 403s scrapers — numbers corroborated at https://factorhouse.io/articles/netflix-kafka-architecture/, secondary.) https://netflixtechblog.com/kafka-inside-keystone-pipeline-dd5aeabaf6bb
- "Disaster Recovery for Multi-Region Kafka at Uber" — Uber Engineering, 2020. "Trillions of messages, petabytes per day" claim plus multi-region replication design. (Primary engineering post.) https://www.uber.com/blog/kafka/
- "Exactly-Once Semantics Are Possible: Here's How Kafka Does It" — Neha Narkhede, Confluent, 2017. Idempotent producer + transactions, the 0.11 release. (Primary, by Kafka co-creator.) https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- Apache Kafka 4.0 release announcement — Apache Software Foundation, 2025. ZooKeeper removal, KRaft-only operation. (Primary.) https://kafka.apache.org/blog#apache_kafka_400_release_announcement
- RabbitMQ documentation — classic queues vs Streams, AMQP model. For the smart-broker contrast. (Primary documentation.) https://www.rabbitmq.com/docs/streams
