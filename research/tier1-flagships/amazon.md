# Amazon (retail platform & Dynamo)

## One-line hook
Amazon decided a shopping cart must *never* refuse an item — and accepting that
"always writeable" requirement forced it to give up strong consistency, producing
the Dynamo paper that launched the entire NoSQL era.

## The core problem
Run a store that is always open. Two intertwined hard problems:

- **Organizational scale.** Around 2000, thousands of engineers changing one giant
  codebase and one set of shared Oracle databases were grinding each other to a halt.
  The bottleneck was coordination, not hardware.
- **Availability at datacenter scale.** With tens of millions of components, something
  is *always* failing — disks, switches, whole datacenters (the Dynamo paper treats
  this as the normal case, not the exception). Yet a checkout that errors out is
  directly lost revenue and eroded trust.

Amazon's distinctive twist: performance targets are stated at the **99.9th
percentile**, not the average, because the slowest requests tend to belong to the
customers with the longest history and the fullest carts (Dynamo paper, 2007).
Traditional RDBMS replication chooses consistency over availability during network
partitions; Amazon's cart needed exactly the opposite trade.

## Architecture overview
**From monolith to services (the org + code story):** Late-90s Amazon.com was a
monolithic C application ("Obidos") in front of big shared Oracle databases. Around
2002 Jeff Bezos issued the famous **API mandate**, recounted publicly in Steve
Yegge's 2011 "platforms rant":

1. All teams expose their data and functionality through service interfaces.
2. Teams communicate *only* through those interfaces.
3. No other interprocess communication: no shared databases, no direct reads of
   another team's store, no back-doors.
4. Technology choice inside a service is free; the interface is the contract.
5. All interfaces must be designed as if they will someday be exposed externally.
6. "Anyone who doesn't do this will be fired."

This turned Amazon into a mesh of small, independently deployed services owned by
"two-pizza teams" — and made AWS (S3 and EC2, 2006) a natural externalization of
what were already internal platform services. Yegge's rant is candid about the cost:
years spent inventing service discovery, monitoring, quotas, and debugging across
hundreds of service boundaries.

**Page render and order pipeline (high level, published):** Rendering one amazon.com
page is an aggregation problem — a front-end request fans out to roughly **100–150
services** (Werner Vogels, ACM Queue interview, 2006): catalog, pricing,
recommendations/personalization, reviews, cart, ads — and composes their responses.
Placing an order is a pipeline, not a transaction: cart → checkout/order capture →
payment authorization → an asynchronous, queue-driven fulfillment workflow
(inventory allocation, warehouse picking/packing, shipping, notifications). Each
stage retries independently instead of holding one giant distributed transaction
open across warehouses. By 2019 the consumer business had moved this entire estate
off Oracle: ~7,500 Oracle databases and 75 PB migrated to DynamoDB, Aurora,
Redshift and friends (AWS blog, 2019).

**Dynamo — the always-on key-value store (SOSP 2007):** built for services like the
shopping cart and session store that need only primary-key get/put but demand
extreme availability.

- Data is partitioned and replicated around a **consistent-hashing ring** with
  virtual nodes; each key is stored on N successor nodes (typically N=3).
- Reads and writes use a **quorum-style** scheme (R + W > N, typically R=W=2), but
  the quorum is **"sloppy"**: if the proper replicas are unreachable, the first N
  *healthy* nodes accept the operation instead.
- Divergent versions are tracked with **vector clocks** and surfaced to the
  application on read; the cart resolves conflicts by merging (union of items).
- **Hinted handoff** delivers misplaced writes back to their true owners on
  recovery; **Merkle trees** let replicas compare and repair drift cheaply in the
  background; a **gossip protocol** spreads membership and failure info so there is
  no master and no single point of failure.

**Component list (plain text):**
- Front-end page-assembly layer (fan-out to ~100–150 services per page, 2006)
- Product catalog, pricing, recommendations, reviews, ads services
- Shopping cart + session services (Dynamo-backed in the paper era)
- Order pipeline: checkout → payment → queued, retryable fulfillment workflow
- Dynamo ring: consistent hashing + virtual nodes; N/R/W sloppy quorums;
  vector clocks; hinted handoff; Merkle-tree anti-entropy; gossip membership
- DynamoDB (2012→): request routers, partition metadata, Multi-Paxos-replicated
  storage partitions (the managed descendant)
- Modern AWS substrate visible in Prime Day posts: EC2/Graviton, Lambda, SQS,
  Aurora, ElastiCache, SageMaker

## Signature ideas
- **The API mandate — architecture as org design.** Bezos's 2002 edict made every
  team a platform with a networked interface and zero shared state. It is Conway's
  Law wielded deliberately: independent services → independent teams → independent
  scaling, deployment, and failure domains. Painful for years, but it is the reason
  Amazon could later sell its internals as AWS (Yegge rant, 2011).
- **SLAs at the 99.9th percentile.** Internal contracts read like "300 ms for 99.9%
  of requests at 500 requests/sec." Averages hide exactly the heavy, high-value
  customers you most want to keep. This framing from the Dynamo paper (2007)
  helped push the whole industry from mean latency to percentile/tail thinking.
- **Consistent hashing with virtual nodes.** Hash both servers and keys onto a ring;
  a key belongs to the next N servers clockwise. Adding or removing a server moves
  only a small arc of keys instead of rehashing everything. Virtual nodes — each
  physical server appearing many times on the ring — smooth the load and let bigger
  machines take proportionally bigger shares (Dynamo paper, 2007).
- **Sloppy quorum + hinted handoff — "always writeable," made real.** A strict
  quorum rejects writes when replicas are unreachable; Dynamo instead writes to the
  first N healthy nodes, with stand-ins storing a "hint" naming the intended owner
  and delivering the data home when it recovers. Your add-to-cart succeeds even
  mid-outage; the cost is temporary divergence. This is the CAP trade-off made
  concrete: availability chosen, consistency relaxed to eventual (Dynamo paper, 2007).
- **Vector clocks + application-level merge.** Each object version carries a list of
  per-node counters; comparing lists reveals whether one version descends from the
  other or they genuinely conflict. Dynamo returns *all* conflicting versions and
  lets the application reconcile — for the cart, merge = union, so an "add" is never
  lost (though a deleted item can occasionally resurface, a quirk the paper admits).
  Deep lesson: only the application knows what "merge" means (Dynamo paper, 2007).
- **Productization: Dynamo → DynamoDB.** Dynamo was software each team had to
  operate; DynamoDB (launched January 2012) is the fully managed, multi-tenant
  descendant. Strikingly, it *dropped* the exotic parts — no sloppy quorum, no
  vector clocks — using leader-based Multi-Paxos replication per partition instead,
  because operability and predictable consistency beat theoretical elegance
  (Vogels launch post, 2012; DynamoDB paper, USENIX ATC 2022). Dynamo's bigger
  legacy is external: Cassandra (2008), Riak, and Voldemort all descend from the paper.

## Key numbers
- Cart service handled tens of millions of requests and ~3 million checkouts in a
  single day during the 2006 holiday season, with no downtime; platform of tens of
  thousands of servers (2007, Dynamo paper —
  https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- Typical Dynamo-era SLA: 300 ms for 99.9% of requests at 500 req/s
  (2007, Dynamo paper — same URL)
- One amazon.com page render calls ~100–150 services
  (2006, Werner Vogels, ACM Queue — https://queue.acm.org/detail.cfm?id=1142065)
- Oracle exit: ~7,500 Oracle databases and 75 PB migrated; consumer-facing latency
  down ~40%, database admin overhead down ~70% (2019, AWS News Blog —
  https://aws.amazon.com/blogs/aws/migration-complete-amazons-consumer-business-just-turned-off-its-final-oracle-database/)
- DynamoDB peaked at 89.2M requests/sec during Prime Day 2021, still single-digit
  milliseconds (2022, DynamoDB paper, USENIX ATC —
  https://www.usenix.org/conference/atc22/presentation/elhemali)
- Prime Day 2023: DynamoDB peaked at 126M req/s; SQS peaked at 86M messages/sec
  (2023, AWS News Blog —
  https://aws.amazon.com/blogs/aws/prime-day-2023-powered-by-aws-all-the-numbers/)
- Prime Day 2024: DynamoDB peaked at 146M req/s; 6,311 Aurora instances processed
  >376B transactions (2024, AWS News Blog —
  https://aws.amazon.com/blogs/aws/how-aws-powered-prime-day-2024-for-record-breaking-sales/)
- Prime Day 2025: DynamoDB peaked at 151M req/s; Lambda 1.7 trillion
  invocations/day; ElastiCache >1.5 quadrillion requests/day; SageMaker 626B
  inference requests; Graviton powered 40% of Amazon's EC2 compute
  (2025, AWS News Blog —
  https://aws.amazon.com/blogs/aws/aws-services-scale-to-new-heights-for-prime-day-2025-key-metrics-and-milestones/)

## Evolution timeline
- **1995–2000** — Monolith era: "Obidos" C application + shared Oracle databases;
  scaling by buying bigger boxes.
- **~2002** — Bezos API mandate; multi-year decomposition into services owned by
  two-pizza teams; internal platform tooling (discovery, monitoring) built from scratch.
- **2004–2006** — Holiday-season database availability pain motivates a homegrown
  store; Dynamo is built and deployed for cart/session-class workloads. AWS launches
  S3 and EC2 (2006), externalizing the platform mindset.
- **2007** — Dynamo paper published at SOSP; ignites the NoSQL movement
  (Cassandra 2008, Riak, Voldemort).
- **2012** — DynamoDB launches as a fully managed service, abandoning the leaderless
  design for Paxos-replicated partitions.
- **2014–2019** — Serverless and event-driven patterns spread (Lambda 2014, SQS/
  Step Functions in the order path); retail estate completes its Oracle exit (2019:
  7,500 databases, 75 PB).
- **2021–2025** — Published Prime Day telemetry tracks the descendant stack at full
  stretch: DynamoDB 89M (2021) → 126M (2023) → 146M (2024) → 151M req/s (2025).

## Visualization hooks
- **The mandate, before/after.** Left: a tangled monolith blob, teams reaching into
  each other's databases, arrows everywhere. Right: clean boxes each with a single
  port, all arrows through APIs. Footer: "no back-doors; anyone who doesn't do this
  will be fired."
- **The hash ring.** Keys and servers on a circle; drop in a new server and watch
  only one arc of keys migrate — side-by-side with naive modulo hashing where
  *everything* reshuffles. Virtual nodes as many small dots sharing a color.
- **Sloppy quorum during a failure.** N=3 replicas of "your cart"; one replica's
  rack dies mid-add-to-cart; the write lands on a stand-in node wearing a luggage
  tag (the hint); when the rack returns, the tagged write walks home.
- **Vector clocks as a family tree.** A network partition splits the cart; the two
  sides' version vectors diverge ([A:2] vs [A:1,B:1]); a later read collects both
  branches and merges to the union — including the famous quirk of the deleted item
  that comes back.
- **The 99.9th percentile.** A latency histogram whose average looks great; zoom
  into the right tail and reveal the biggest carts living there.
- **One page, 150 calls.** A product page exploding into a starburst of ~150 service
  calls (price, stock, reviews, recs...), each with a tiny SLA badge, reassembling
  into one page in under a second.
- **Prime Day dial.** A speedometer sweeping to 151,000,000 requests per second
  (2025) — caption: "roughly every human on Earth pressing a button every
  53 milliseconds."
- **Dynamo family tree.** The 2007 paper branching into Cassandra, Riak, Voldemort,
  and DynamoDB — with DynamoDB's branch visibly pruning off vector clocks and
  sloppy quorum on its way to becoming a managed cloud service.

## Sources
- "Dynamo: Amazon's Highly Available Key-value Store" — DeCandia et al., SOSP 2007 —
  the core paper: 99.9th-percentile SLAs, consistent hashing, vector clocks, sloppy
  quorum, hinted handoff, Merkle trees, gossip; cart war stories.
  https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf (primary, paper)
- "Amazon's Dynamo" — Werner Vogels, All Things Distributed, 2007 — the CTO's
  framing of why the system and paper exist.
  https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html (primary, official blog)
- "Stevey's Google Platforms Rant" — Steve Yegge, 2011 (preserved gist) — firsthand
  recounting of the 2002 Bezos API mandate and Amazon's SOA transformation.
  https://gist.github.com/chitchcock/1281611 (primary-adjacent memoir; the mandate
  text is Yegge's from-memory paraphrase, not a leaked document)
- "A Conversation with Werner Vogels" — ACM Queue, 2006 — service-oriented Amazon,
  100–150 services per page, "you build it, you run it."
  https://queue.acm.org/detail.cfm?id=1142065 (primary, interview)
- "Amazon DynamoDB – a Fast and Scalable NoSQL Database Service Designed for
  Internet Scale Applications" — Werner Vogels, All Things Distributed, 2012 —
  launch post; lessons from operating Dynamo and why managed won.
  https://www.allthingsdistributed.com/2012/01/amazon-dynamodb.html (primary, official blog)
- "Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL
  Database Service" — Elhemali et al., USENIX ATC 2022 — how DynamoDB actually
  works (Paxos partitions, adaptive capacity, admission control); Prime Day 2021
  figure. https://www.usenix.org/conference/atc22/presentation/elhemali (primary, paper)
- "Migration Complete – Amazon's Consumer Business Just Turned off its Final Oracle
  Database" — Jeff Barr, AWS News Blog, 2019 — 7,500 DBs / 75 PB Oracle exit.
  https://aws.amazon.com/blogs/aws/migration-complete-amazons-consumer-business-just-turned-off-its-final-oracle-database/
  (primary, official blog)
- "Prime Day 2023 – Powered by AWS, All the Numbers" — AWS News Blog, 2023 —
  yearly scale telemetry.
  https://aws.amazon.com/blogs/aws/prime-day-2023-powered-by-aws-all-the-numbers/
  (primary, official blog)
- "How AWS powered Prime Day 2024 for record-breaking sales" — AWS News Blog,
  2024 — DynamoDB 146M req/s, Aurora figures.
  https://aws.amazon.com/blogs/aws/how-aws-powered-prime-day-2024-for-record-breaking-sales/
  (primary, official blog)
- "AWS services scale to new heights for Prime Day 2025: key metrics and
  milestones" — AWS News Blog, 2025 — DynamoDB 151M req/s and the rest of the
  2025 telemetry.
  https://aws.amazon.com/blogs/aws/aws-services-scale-to-new-heights-for-prime-day-2025-key-metrics-and-milestones/
  (primary, official blog)
- "Jeff Bezos' API mandate" — Axway blog, accessed 2026 — clean secondary summary
  of the mandate's rules for quick reference.
  https://blog.axway.com/learning-center/digital-strategy/api-first/jeff-bezos-api-mandate
  (secondary)
