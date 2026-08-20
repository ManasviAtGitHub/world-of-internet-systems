# Airbnb

## One-line hook
A marketplace where every item is unique and sold by the night: Airbnb's story is a decade-long
escape from one Ruby monolith ("Monorail") into services — while search ranking became a machine
learning system and payments became a graph of currencies that must never double-charge.

## The core problem
Match a traveler to a one-of-a-kind home for specific dates, in a two-sided marketplace spanning
191 countries and 70+ currencies. Unlike e-commerce, no item is ever consumed twice and each
listing can host only one booking per date — so search, ranking, availability, and payments are
all entangled with a calendar. Meanwhile the engineering org grew from ~200 to ~1,000 engineers
(2015–2018) all committing to a single Rails monolith, where reverts and rollbacks cost engineers
~15 hours per week — an organizational scaling problem as much as a technical one.

## Architecture overview
End-to-end data flow, from a search to a paid booking:

1. **Search request.** The client hits the API gateway, which routes to search services. In the
   SOA design, services are typed: **data services** own reads/writes to a single data source,
   **derived-data services** build on those, **middle-tier** services hold business logic, and
   **presentation services** assemble UI-facing responses. Requests fan out from the gateway
   through these layers.
2. **Retrieval.** Candidate listings are filtered by location, dates (availability calendar),
   guest count, and filters. Since ~2025 an embedding-based retrieval (EBR) stage also runs: a
   two-tower neural model (listing tower computed offline daily; query tower evaluated in real
   time) with an IVF approximate-nearest-neighbor index — chosen over HNSW because listings
   update constantly and queries are heavily geo-filtered — narrows millions of homes to a
   scoreable pool.
3. **Ranking.** Heavier ML models score the pool. This system evolved publicly: hand-tuned
   scoring → gradient-boosted decision trees → deep networks (KDD 2018/2019 papers), with
   listing embeddings learned from click sessions powering real-time personalization, and
   sequence models over the whole "guest journey" by 2026. Ranking must optimize both sides:
   guest preference and host acceptance.
4. **Booking write path.** Checkout calls reservation/booking services which validate the
   requested nights against the listing's availability calendar and commit the reservation while
   marking those dates — the one place the system must be strongly consistent, since two guests
   can race for the same nights (Airbnb has published little primary detail here; see note in
   Sources).
5. **Payments.** The payments SOA collects from the guest in their currency, holds, then pays
   out to the host in theirs — any currency pair, modeled as a complete directed graph of
   corridors across 20+ processors. Every money movement is recorded in double-entry style
   (every cent has a clear source and destination), and all payment mutations go through
   **Orpheus**, their idempotency framework: each request carries an idempotency key and is
   split into pre-RPC / RPC / post-RPC phases with the key store sharded — so client retries can
   never double-charge.
6. **Events & data.** Services publish mutations as standard events; **SpinalTap** (open-sourced)
   streams database change-capture into Kafka. Offline, thousands of daily pipeline tasks are
   orchestrated by **Airflow** — invented at Airbnb — feeding analytics, ML training, and
   financial reporting.
7. **Service-to-service plumbing.** Discovery/routing began with **SmartStack** (2013): a
   Nerve sidecar health-checks each service and registers it in ZooKeeper; a Synapse sidecar
   watches ZooKeeper and rewrites a local HAProxy config, so callers just hit localhost. From
   2019 Airbnb rebuilt this as **AirMesh**, an Istio/Envoy service mesh, alongside the move from
   EC2 instances to Kubernetes. Services speak Thrift-defined IDL interfaces with generated
   clients.

Component list (plain text):
- Clients -> API gateway -> presentation / middle-tier / derived-data / data services
- Search: retrieval (geo+date filter, EBR two-tower + IVF ANN) -> ML ranker (GBDT->DNN->sequence models)
- Availability calendar + reservation/booking services (strongly consistent write path)
- Payments SOA: collection, payout, currency-corridor graph, double-entry ledger, Orpheus idempotency
- SmartStack (Nerve+Synapse+ZooKeeper+HAProxy) -> AirMesh (Istio/Envoy on Kubernetes)
- SpinalTap CDC -> Kafka event bus
- Airflow-orchestrated offline data platform (Hadoop/warehouse, ML training, financial reporting)

## Signature ideas
- **Monolith → SOA by 1% comparison.** Rather than a big-bang rewrite, each migrated read path
  ran dual: send 1% of traffic down the new service, compare responses against Monorail
  (idempotent reads compared directly; writes validated via shadow databases; Diffy replayed
  production traffic), then ratchet to 100%. The migration was as much process as architecture:
  four tenets, including "services own reads/writes to their data" and "publish mutations as
  standard events."
- **SmartStack: the service mesh before the name existed.** Out-of-process sidecars (Nerve
  registers health into ZooKeeper; Synapse renders HAProxy config locally) gave every service
  discovery, load balancing, and failover without touching application code — in 2013. Its
  successor AirMesh (Istio/Envoy) is the same idea with a modern control plane.
- **Airflow: workflows as code.** By 2015 Airbnb ran 5,000–6,000 Hadoop tasks a day and existing
  schedulers (Oozie, Azkaban) didn't fit. Maxime Beauchemin's insight: pipeline authors are data
  people, not systems engineers — give them DAGs defined in Python (loops, parameterization,
  code review) plus a scheduler and a UI showing dependencies, ownership, and retries. It became
  the industry-standard orchestrator (Apache Airflow).
- **Search ranking as a marketplace problem.** Airbnb's ranking papers stress what makes them
  unusual: one guest per listing per dates, no repeated consumption, and two-sided optimization
  (will the host accept?). Their published arc — GBDT → deep learning (KDD 2018/2019), listing
  embeddings from search sessions, then two-tower embedding retrieval with IVF tuned around
  Euclidean distance for balanced clusters — is one of the best-documented ML-system evolutions
  anywhere.
- **Orpheus: idempotency as a library.** Payments requests are wrapped in a three-phase protocol
  (pre-RPC records intent, RPC talks to the processor, post-RPC records outcome) keyed by a
  client-supplied idempotency key, with the idempotency store sharded by key and reads pinned to
  the master to dodge replica lag. Product engineers get "exactly-once effect" retries for free.
- **Money across a currency graph with a double-entry spine.** Guest-side and host-side
  transactions form a complete directed graph over 70+ currencies (with self-loops for same
  currency), routed over 20+ processors; financial pipelines enforce "Double Entry Satisfied" —
  every cent traceable from source to destination — so books balance at global scale.
- **The calendar-consistency problem.** Availability is the marketplace's inventory: the booking
  path must make double-booking structurally impossible (effectively a uniqueness guarantee on
  listing+date under concurrency), while hosts sync external calendars via slow iCal polling —
  a neat contrast between in-platform strong consistency and cross-platform eventual
  consistency. (Well-known as a design problem; thin on Airbnb-primary detail.)

## Key numbers
- Engineering org grew ~200 → ~1,000 engineers, 2015–2018; ~15 hours/engineer/week lost to
  reverts/rollbacks in the monolith; some page loads up to 10x faster after migration (InfoQ,
  2019) — https://www.infoq.com/news/2019/02/airbnb-monolith-migration-soa/
- Payments platform: 191 countries, 70+ currencies, 20+ payment processors / two dozen payment
  routes ("Scaling Airbnb's Payment Platform," Airbnb Tech Blog, c. 2018–2019) —
  https://medium.com/airbnb-engineering/scaling-airbnbs-payment-platform-43ebfc99b324
- 5,000–6,000 Hadoop tasks/day when Airflow was created; first production deploy on six AWS
  c3.8xlarge nodes (InformationWeek, 2015) —
  https://www.informationweek.com/machine-learning-ai/airbnb-creates-hadoop-workflow-system-airflow
- 5M+ hosts and 8M+ active listings (Airbnb official figures as of Q3 2024, via aggregator) —
  https://www.demandsage.com/airbnb-statistics/
- EBR launched in production with statistically significant booking gains (Airbnb Tech Blog,
  2025) — https://airbnb.tech/uncategorized/embedding-based-retrieval-for-airbnb-search/
- SOA-era stack migrated from EC2 to Kubernetes; Thrift IDL for service interfaces (QCon SF
  2018 talk, reported 2019) — https://www.infoq.com/news/2019/02/airbnb-monolith-migration-soa/

## Evolution timeline
- **2008–2013** — Rails monolith ("Monorail") does everything; MySQL underneath.
- **2013** — SmartStack (Nerve + Synapse) open-sourced: sidecar service discovery for the first
  satellite services.
- **2014–2015** — Airflow built internally, open-sourced 2015 (Apache project thereafter).
- **2015–2018** — Hypergrowth (200→1,000 engineers) makes Monorail the bottleneck; SOA migration
  runs via incremental traffic comparison; SpinalTap CDC; Thrift service standards; QCon SF 2018
  talk documents it.
- **2018–2019** — ML search ranking era: listing embeddings and deep-learning ranking published
  at KDD; payments platform and Orpheus idempotency published.
- **2019–2023** — Kubernetes migration; AirMesh service mesh on Istio/Envoy replaces SmartStack.
- **2025–2026** — Embedding-based retrieval (two-tower + IVF) in production; sequence-model
  ranking ("guest journey" / JourneyFormer at KDD 2026).

## Visualization hooks
1. **Monorail fission.** One massive block labeled Monorail cracking into typed service shapes
   (data / derived-data / middle-tier / presentation) under an API gateway — with a traffic
   slider animating 1% → 10% → 100% flowing down the new path while a comparator checks both
   answers match.
2. **The sidecar trio.** A service box flanked by two small sidecars: Nerve pulsing heartbeats up
   to ZooKeeper, Synapse pulling the registry down into a local HAProxy — then morph the same
   picture into Envoy/Istio to show "the mesh was always the same idea."
3. **The search funnel over a map.** Millions of listing dots → geo/date filter shrinks them →
   EBR cone (two towers projecting query and listings into the same vector space) → ranker
   reorders the final page. Emphasize dates as a filter dimension no e-commerce funnel has.
4. **The currency graph.** Currency nodes (USD, EUR, JPY, CAD...) with directed edges as payment
   corridors; animate one booking: CAD leaves a guest, hops through Airbnb's ledger (double-entry
   entries appearing in pairs), lands as AUD with a host.
5. **The retry that couldn't double-charge.** A flaky network hammers "Pay" five times; five
   arrows hit the Orpheus wall keyed by one idempotency key; exactly one passes to the processor;
   the other four get the recorded response.
6. **Two guests, one calendar.** A month-grid calendar; two cursors race to book the same nights;
   one transaction locks the dates, the other bounces — then zoom out to show a slow iCal sync
   loop with an external platform, where the race can't be locked away.
7. **A DAG wakes up at midnight.** An Airflow DAG rendered as a constellation: upstream tasks
   light up and cascade downstream; one node fails and glows red; retry, backfill, and the
   ownership label — pipelines as living code.
8. **Two-sided ranking.** A guest card and a host card on a balance scale: ranking must weigh
   "will the guest book it?" against "will the host accept?" — the marketplace twist in one
   image.

## Sources
- "Airbnb's Great Migration: From Monolith to Service-Oriented" — Jessica Tai, QCon SF 2018
  (talk page). https://qconsf.com/sf2018/sf2018/presentation/airbnbs-great-migration-monolith-service-oriented.html
  — The primary migration narrative. Primary.
- "Airbnb Monolith Migration to SOA" — InfoQ news summary, 2019.
  https://www.infoq.com/news/2019/02/airbnb-monolith-migration-soa/ — Numbers (200→1,000
  engineers, 15 hrs/week, 10x pages), tenets, SpinalTap/Diffy, Kubernetes. Secondary (reliable
  summary of the primary talk).
- "Scaling Airbnb's Payment Platform" — Airbnb Tech Blog (Medium), c. 2018–2019.
  https://medium.com/airbnb-engineering/scaling-airbnbs-payment-platform-43ebfc99b324 — Currency
  graph, 191 countries / 70+ currencies / 20+ processors, billing SOA. Primary. (Medium blocks
  some automated fetchers; figures cross-checked via search snippets.)
- "Avoiding double payments in a distributed payments system" — Jon Chew, Airbnb Tech Blog, 2019.
  https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb
  — Orpheus, three-phase idempotency, sharded key store. Primary.
- "Tracking the money — Scaling financial reporting at Airbnb" — Alice Liang, Airbnb Tech Blog.
  https://medium.com/airbnb-engineering/tracking-the-money-scaling-financial-reporting-at-airbnb-6d742b80f040
  — Double-entry principle in the financial pipeline. Primary.
- "Airflow: a workflow management platform" — Maxime Beauchemin, Airbnb (nerds.airbnb.com /
  Medium), 2015. http://nerds.airbnb.com/airflow/ — The original announcement; workflows-as-code
  rationale. Primary.
- "Airbnb Creates Hadoop Workflow System: Airflow" — InformationWeek, 2015.
  https://www.informationweek.com/machine-learning-ai/airbnb-creates-hadoop-workflow-system-airflow
  — 5–6k tasks/day, Oozie/Azkaban rejection, deployment details. Secondary (interview with
  Beauchemin).
- SmartStack: Synapse repository — Airbnb, GitHub (from 2013).
  https://github.com/airbnb/synapse — Nerve/Synapse mechanics. Primary (code + README).
- "Airbnb Service Discovery: Past, Present, Future" — Chase Childers, KubeCon NA 2019 (session).
  https://kccncna19.sched.com/event/UaVz/ — SmartStack→mesh evolution. Primary (talk listing).
- "Improving Istio Propagation Delay" — Ying Zhu, Airbnb Tech Blog, 2023.
  https://airbnb.tech/infrastructure/improving-istio-propagation-delay/ — Confirms AirMesh =
  Istio/Envoy in production. Primary.
- "Real-time Personalization using Embeddings for Search Ranking at Airbnb" — Grbovic & Cheng,
  KDD 2018. https://www.kdd.org/kdd2018/accepted-papers/view/real-time-personalization-using-embeddings-for-search-ranking-at-airbnb
  — Listing embeddings; marketplace framing. Primary (peer-reviewed; blog version:
  https://medium.com/airbnb-engineering/listing-embeddings-for-similar-listing-recommendations-and-real-time-personalization-in-search-601172f7603e)
- "Applying Deep Learning to Airbnb Search" — Haldar et al., 2018 (KDD 2019).
  https://arxiv.org/abs/1810.09591 — The GBDT→NN journey, candid lessons. Primary.
- "Embedding-Based Retrieval for Airbnb Search" — Airbnb Tech Blog, 2025.
  https://airbnb.tech/uncategorized/embedding-based-retrieval-for-airbnb-search/ — Two-tower +
  IVF retrieval stage. Primary.
- "Airbnb Statistics" — DemandSage aggregator, 2024–2026.
  https://www.demandsage.com/airbnb-statistics/ — 5M+ hosts / 8M+ listings figures sourced from
  Airbnb's official results. Secondary (aggregator of primary filings; verify against
  news.airbnb.com before publishing).
- NOTE ON THE CALENDAR PROBLEM: Airbnb has published little first-party engineering detail on
  booking/calendar concurrency; treatments like systemdesign.academy's "Design Airbnb"
  (https://www.systemdesign.academy/interview/design-airbnb) are interview-style reconstructions.
  Label calendar-consistency content as "inferred design," not Airbnb-documented fact.
