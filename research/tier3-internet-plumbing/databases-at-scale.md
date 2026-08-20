# Databases at Scale

## One-line hook
Every hypergrowth company climbs the same ladder — cache, replicate, partition, shard — and each rung quietly trades away another piece of the comforting single-machine illusion (Figma bought four years of runway before paying the sharding toll; Instagram paid it with a two-person infra team).

## The core problem
A database on one machine has three hard ceilings: it can only serve so many queries, hold so much data, and it is one power cable away from total outage. Scaling reads is comparatively easy (copies + caches). The brutal problems are scaling *writes* and surviving *failures* — because both force you to keep multiple machines agreeing about the same data over a network that can delay, drop, or partition messages. Everything in this topic — replication lag, shard keys, CAP, Raft — is a different face of that one problem: **agreement over an unreliable network**.

## How it works
The standard scaling ladder, rung by rung:

1. **One box + indexes + vertical scaling.** Works far longer than people admit. Figma ran a single Postgres instance on AWS's largest machine until ~2020.
2. **Caching.** Put memcached/Redis in front; absorb repeated reads. Cheap, but introduces invalidation bugs and stampedes — and does nothing for writes.
3. **Read replicas (leader–follower replication).** All writes go to one leader; the leader streams its change log to followers (Postgres ships WAL records; MySQL ships binlog events, row- or statement-based). Reads fan out to followers.
   - **Async (the default in MySQL and Postgres):** the leader commits without waiting for followers → fast, but a crashed leader may take acknowledged-but-unreplicated writes with it, and followers serve *stale* reads (replication lag).
   - **Sync / semi-sync:** the leader waits for one or more replicas to confirm (Postgres `synchronous_commit`/`synchronous_standby_names`, MySQL semisync plugin) → no lost acknowledged writes, at latency cost.
   - Lag causes real anomalies: read-your-own-writes violations (you post a comment, refresh, it's gone), time-travel between successive page loads. Apps route "must be fresh" reads to the leader.
4. **Vertical partitioning.** Move whole tables/domains ("users DB", "files DB") to separate databases. Figma did this 2020–2022, reaching a dozen vertically partitioned databases plus caching and replicas before true sharding.
5. **Horizontal sharding.** Split *rows of one table* across machines by a shard key:
   - **Range sharding:** key ranges per shard (Bigtable/HBase style). Great for range scans; risk of hot ranges (e.g., time-ordered keys hammering the last shard).
   - **Hash sharding:** hash(key) → shard. Even spread; kills cross-shard range queries. Figma hashes its shard keys (UserID, FileID, OrgID).
   - **Directory/lookup sharding:** an explicit mapping table from key → shard; maximally flexible, one more thing to keep consistent.
   - **Logical vs physical shards — the universal trick:** create many more logical shards than machines (Instagram: several thousand logical shards as Postgres schemas; Notion: 480 logical shards on 32 physical hosts, later respread across 96 with zero routing changes). Resharding becomes "move logical shards," not "rewrite hash function."
   - The price: cross-shard joins and transactions mostly stop working; a query router (Vitess's vtgate, Figma's DBProxy) parses SQL, scatters/gathers across shards, and hides the mess from application code.
6. **Consensus for the control plane (Raft intuition).** Someone must decide *who is leader* without split-brain. Raft: nodes are followers until a randomized election timeout fires; a candidate requests votes and wins with a majority for that term; leaders replicate a log and commit entries once a majority stores them. Majorities intersect, so two leaders can't both commit for the same term — this is the machinery behind etcd, CockroachDB ranges, MongoDB replica-set elections, and Kafka's KRaft.

Concrete end-to-end flow (sharded read): app asks for file 8123 → DBProxy/vtgate extracts FileID, hashes it → logical shard 371 → routing table says shard 371 lives on physical DB 14 → query rewritten and sent to DB 14's leader (or a replica if staleness is acceptable) → row returns; the application never learns sharding exists.

Plain-text component list:
- Leader database + WAL/binlog shipping → N read replicas (async or semi-sync)
- Cache tier (with invalidation strategy) in front of reads
- Shard router / query engine (vtgate, DBProxy) + shard topology metadata
- Logical shards (schemas/keyspaces) mapped onto physical hosts
- Consensus service (Raft group) holding topology + electing leaders
- Backfill/migration machinery for resharding (double-write or log-replay + verify + cutover)

## Signature ideas
- **Replication lag is a product problem, not just an ops metric.** Async replication means every replica is a time machine set slightly in the past. The classic fixes — sticky reads to the leader after a write, monotonic session tokens, bounded-staleness reads — are all ways of buying back "read your own writes" without paying synchronous-commit latency everywhere.
- **CAP, stated correctly.** In Gilbert & Lynch's proof (2002) of Brewer's conjecture (2000): a system cannot simultaneously guarantee linearizable consistency (C), availability of every request to every non-failed node (A), while tolerating arbitrary message loss (P). Since real networks partition, P is not optional — during a partition you choose C (refuse/stall some requests) or A (answer, possibly stale/divergent). "CA" is not a thing you can deploy. Crucially, CAP says *nothing* about behavior when the network is healthy — and Kleppmann argues the C/A/P labels are too blunt to classify real systems at all.
- **PACELC completes the picture.** Abadi (IEEE Computer, 2012): if Partitioned, trade Availability vs Consistency; Else (normal operation), trade Latency vs Consistency. This explains why Dynamo-style stores are eventually consistent even when nothing is broken — they're paying for low latency, not for partition tolerance.
- **Majorities are magic (Raft in one breath).** Any two majorities of the same cluster share at least one node. Raft exploits this twice — elections (a candidate needs a majority, and voters refuse candidates with stale logs) and commits (an entry on a majority can't be lost) — so leadership changes never silently drop committed writes. Randomized election timeouts are the delightfully low-tech trick that prevents endless split votes.
- **Resharding is the boss fight.** Choosing a shard key is forever-ish: it decides which queries stay single-shard (fast) and which scatter. Migrations follow a now-standard choreography — start double-writes or subscribe to the change log, backfill history, verify checksums, then cut over — and the best teams make cutover boring: Figma's first sharded-table failover cost about ten seconds of partial availability (2024); Vitess turned this choreography into a product feature (VReplication).
- **Jepsen: CAP-in-practice audits.** Kyle Kingsbury's Jepsen project partitions real databases and checks their guarantees. Findings routinely embarrass marketing: MongoDB 4.2.6 lost committed writes and exhibited anomalies even at the strongest read/write concerns (2020); even PostgreSQL 12.3's "serializable" isolation allowed a class of anomaly, fixed after the report (2020). Moral: consistency claims are empirical claims.

## Key numbers
- Instagram at sharding time: ~25 photos and 90 likes per *second*, several thousand logical shards, 64-bit IDs minting up to 1024 IDs per shard per millisecond (2011) — https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
- Notion: 480 logical shards across 32 physical Postgres hosts (2021), later respread to 96 hosts with no routing-logic change — https://www.notion.com/blog/sharding-postgres-at-notion
- Figma: ~100x database growth since 2020; nine-month horizontal-sharding project; first sharded table shipped Sept 2023 with ~10 seconds of partial availability; tables of several TB / billions of rows (2024) — https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/
- Vitess: created at YouTube in 2010, open-sourced 2011, grew to tens of thousands of MySQL nodes at YouTube; CNCF's 8th graduated project, Nov 5, 2019 — https://vitess.io/docs/overview/history/ and https://www.cncf.io/announcements/2019/11/05/cloud-native-computing-foundation-announces-vitess-graduation/
- CAP conjecture 2000 (Brewer, PODC keynote); proof 2002 (Gilbert & Lynch, SIGACT News) — https://dl.acm.org/doi/10.1145/564585.564601
- PACELC formulated 2012 (Abadi, IEEE Computer) — https://ieeexplore.ieee.org/document/6127847/
- Raft published 2014 (Ongaro & Ousterhout, USENIX ATC, "In Search of an Understandable Consensus Algorithm") — https://raft.github.io/raft.pdf
- Jepsen: MongoDB 4.2.6 failed to preserve snapshot isolation even at strongest settings (2020) — https://jepsen.io/analyses/mongodb-4.2.6
- Jepsen: PostgreSQL 12.3 serializable-isolation anomaly found and subsequently patched (2020) — https://jepsen.io/analyses/postgresql-12.3
- MySQL replication is asynchronous by default; semi-sync is an opt-in plugin (current 8.4 docs) — https://dev.mysql.com/doc/refman/8.4/en/replication-semisync.html
- Postgres streaming replication is async by default; `synchronous_standby_names`/`synchronous_commit` enable sync modes (current docs, ch. "High Availability") — https://www.postgresql.org/docs/current/warm-standby.html

## Who uses it and how
- **Instagram (Postgres, 2011–2013):** the canonical tiny-team sharding story — thousands of logical shards as Postgres schemas, Snowflake-inspired 64-bit IDs (timestamp + shard id + per-shard sequence) generated *inside* Postgres with PL/PGSQL, so photos sort by time without a central ID service. Their follow-up lists the pragmatic ladder: partial indexes, memory-fitting working sets, and replication tuning.
- **YouTube → Vitess → everyone (MySQL):** YouTube outgrew a single MySQL leader, built Vitess as a routing/sharding proxy layer so application code stays sharding-unaware, scaled to tens of thousands of nodes, open-sourced it (2011); it graduated CNCF (2019) and now runs MySQL fleets at Slack, Square/Block, GitHub, JD.com, and powers PlanetScale.
- **Figma (Postgres, 2020–2024):** the modern replay of the ladder under 100x growth: biggest-box vertical scaling → caching + read replicas → a dozen vertically partitioned DBs → DBProxy (Go query engine) with hashed shard keys and "colos" (groups of tables sharing a shard key), separating logical from physical sharding so cutovers took seconds. They evaluated CockroachDB/TiDB/Spanner/Vitess and chose to shard vanilla Postgres to avoid a risky wholesale migration.
- **Notion (Postgres, 2021–2023):** sharded by workspace ID into 480 logical shards over 32 machines; chose 480 for its divisibility so future re-spreads (they later went to 96 hosts) move whole logical shards without changing routing math.
- **Amazon Dynamo lineage (2007):** the other branch of the tree — give up single-leader entirely: consistent hashing, every replica accepts writes, vector clocks + read repair reconcile divergence. It's PACELC's "EL" corner made flesh, and the design DNA of DynamoDB, Cassandra, and Riak.

## Visualization hooks
- **The scaling ladder as a literal ladder/game map:** one server → +cache → +replicas → vertical partitions → shards, each rung annotated with what breaks next (invalidation! lag! cross-table joins! cross-shard transactions!) — Figma's timeline maps onto it perfectly.
- **Replication lag time machine:** leader clock at T, replicas at T−200ms and T−3s; a user writes a comment then reads from the lagging replica and watches their own comment vanish; then a "sticky session" arrow pins them to the leader and it reappears.
- **Sync vs async race:** two side-by-side commits — async returns instantly then the leader dies and the write evaporates; sync waits for the follower's ack, survives the same crash. A latency bar vs a durability shield.
- **Hash vs range sharding heat map:** the same insert stream (timestamped keys) hitting range shards (last shard glows red-hot) vs hash shards (even shimmer, but a range query shatters into scatter-gather arrows to every shard).
- **Logical→physical shard remap:** 480 small tiles grouped onto 32 racks; racks double to 64 and tiles slide over in whole units while the hash function on top never changes — the single best argument for logical sharding, fully animatable (Notion's exact numbers).
- **Raft election:** five nodes with visible countdown rings; leader dies; first ring to empty becomes candidate, votes fly, majority crowns it — then a split-vote round where two candidates tie and randomized timeouts break the symmetry. (An interactive classic already exists: https://thesecretlivesofdata.com/raft/ — homage material.)
- **CAP as a fork in the road:** a network partition slices the cluster; left path (choose C) shows minority side refusing writes with errors; right path (choose A) shows both sides accepting writes and the divergence accumulating, then the ugly merge. Overlay PACELC as the "even without the cut" latency dial.
- **Resharding pipeline:** four-stage conveyor — double-write taps on, backfill crawler copies history, checksum comparator turns green, traffic switch flips — with Figma's "10 seconds of partial availability" as the dramatic cutover moment.

## Sources
- "Sharding & IDs at Instagram" — Instagram Engineering, 2011. Logical shards as Postgres schemas, in-database Snowflake-style ID generation, concrete load numbers. (Primary engineering post.) https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
- "Handling Growth with Postgres: 5 Tips From Instagram" — Instagram Engineering, 2013. The pre-sharding toolbox (partial indexes, working set in memory, replication practice). (Primary engineering post.) https://instagram-engineering.com/handling-growth-with-postgres-5-tips-from-instagram-d5d7e7ffdfcb
- "How Figma's Databases Team Lived to Tell the Scale" — Figma blog, March 14, 2024. Full ladder narrative: 100x growth, vertical partitioning, DBProxy, colos, logical vs physical sharding, 9-month timeline, 10-second cutover. (Primary engineering post; best single modern sharding case study.) https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/
- "Herding elephants: lessons learned from sharding Postgres at Notion" — Notion blog, 2021. 480 logical / 32 physical design, shard-key choice, double-write + backfill + verify migration. (Primary engineering post.) https://www.notion.com/blog/sharding-postgres-at-notion
- Vitess history — vitess.io docs; plus CNCF graduation announcement, Nov 5, 2019. YouTube origins, scale, adopters. (Primary documentation / primary foundation announcement.) https://vitess.io/docs/overview/history/ ; https://www.cncf.io/announcements/2019/11/05/cloud-native-computing-foundation-announces-vitess-graduation/
- "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services" — Gilbert & Lynch, ACM SIGACT News, 2002. The CAP proof and its precise definitions. (Primary paper.) https://dl.acm.org/doi/10.1145/564585.564601
- "Consistency Tradeoffs in Modern Distributed Database System Design: CAP is Only Part of the Story" — Daniel Abadi, IEEE Computer, 2012. PACELC. (Primary paper.) https://ieeexplore.ieee.org/document/6127847/
- "Please stop calling databases CP or AP" — Martin Kleppmann, 2015. Why CAP's definitions fit real systems poorly; sharper vocabulary (linearizability, staleness). (Primary essay by the DDIA author; Kleppmann-adjacent spine for the CAP section.) https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html
- "In Search of an Understandable Consensus Algorithm" — Ongaro & Ousterhout, USENIX ATC 2014. Raft: elections, log replication, safety argument. (Primary paper.) https://raft.github.io/raft.pdf
- Jepsen analyses — MongoDB 4.2.6 (2020) and PostgreSQL 12.3 (2020), Kyle Kingsbury. Empirical consistency testing under partitions; the reality check on vendor claims. (Primary independent analyses.) https://jepsen.io/analyses/mongodb-4.2.6 ; https://jepsen.io/analyses/postgresql-12.3
- MySQL 8.4 Reference Manual — Replication chapter (async default, semi-sync plugin, binlog formats). (Primary documentation.) https://dev.mysql.com/doc/refman/8.4/en/replication.html
- PostgreSQL documentation — "High Availability, Load Balancing, and Replication" (WAL shipping, streaming replication, synchronous modes). (Primary documentation.) https://www.postgresql.org/docs/current/high-availability.html
- "Dynamo: Amazon's Highly Available Key-value Store" — DeCandia et al., SOSP 2007. The leaderless/eventually-consistent branch: consistent hashing, vector clocks, sloppy quorums. (Primary paper.) https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
