# Uber

## One-line hook
Uber turned "where is everyone right now?" into a data-structure problem: index the planet as
hexagons (H3), shard the matching engine with a gossip ring (Ringpop), and treat dispatch as a
real-time marketplace that reprices itself cell by cell.

## The core problem
Match a rider to the best nearby driver in seconds, continuously, in thousands of cities — while
every active driver's phone reports its GPS position every ~4 seconds. That is a firehose of
location writes (a 2015 design target was ~1M location writes/sec with far higher read load), a
geospatial nearest-neighbor query that must use road-network ETA rather than straight-line
distance, and a two-sided market whose price must respond to local supply/demand imbalances in
near real time. By 2024 the platform completed ~33 million trips per day. The system must also
survive machine, service, and whole-datacenter failure without dropping in-progress trips.

## Architecture overview
End-to-end data flow, rider taps "Request" to driver assignment:

1. **Driver location pipeline (always running).** The driver app sends a GPS fix roughly every 4
   seconds to Uber's mobile edge/gateway. Updates flow into Kafka topics and into the dispatch
   system's **supply service**, which keeps each driver's state machine (vehicle type, seats,
   available vs. on-trip) and writes the driver's position into a geospatial index. In the 2015
   architecture that index was built on Google S2 cells (cell ID as the sharding key); H3
   hexagons later became Uber's standard grid for marketplace analytics, surge, and dispatch-area
   logic. Map-matching/routing services snap raw GPS to roads and feed the ETA engine.
2. **Request.** The rider taps Request. The **demand service** records the order: product type,
   seat count, special requirements.
3. **Matching (DISCO — "dispatch optimization").** DISCO queries the geospatial supply index with
   a coarse filter — grid cells covering the rider's vicinity (the hex/cell neighborhood) — to
   get candidate drivers, then calls the routing service to compute actual on-road ETAs for each
   candidate. Candidates are ranked by ETA (including drivers about to finish nearby trips), and
   the offer is pushed to the chosen driver's phone. Accept → trip state machine starts;
   decline/timeout → next candidate.
4. **Pricing.** Surge is computed per hexagonal cell per product from the local demand/supply
   ratio (plus historical patterns); the multiplier is baked into the upfront fare the rider sees
   before confirming.
5. **Persistence.** Trip data historically landed in **Schemaless** (Uber's sharded-MySQL,
   append-only datastore, built after a 2014 scaling crisis on Postgres), later evolved into
   **Docstore**. The 2021 fulfillment rewrite moved live order/session state onto Google Cloud
   Spanner for transactional consistency.
6. **Scale-out of DISCO itself.** Dispatch nodes form a **Ringpop** cluster: SWIM gossip for
   membership + consistent hashing to own slices of the workload; any node can receive a request
   and "handle or forward" it to the owner. RPC between services used **TChannel**.
7. **Failure handling.** Everything retryable, everything killable (crash-only). For datacenter
   failover, the dispatch system periodically pushes an encrypted state digest to driver phones;
   after failover, phones replay the digest so in-progress trips survive.

Component list (plain text):
- Mobile apps (rider, driver) -> edge gateway/API
- Kafka (location + event streams; Heatpipe schema validation; uForwarder push-consumer proxy)
- Supply service (driver state machines) / Demand service (rider orders)
- Geospatial index (S2 cells 2015-era; H3 hex grid thereafter)
- Routing / map-matching / ETA services
- DISCO matching engine (Node.js originally), sharded via Ringpop, RPC via TChannel
- Surge pricing service (per-hex supply/demand)
- Datastores: Schemaless -> Docstore (sharded MySQL + Raft); Spanner for fulfillment (2021+)
- Real-time analytics: Kafka -> Flink -> Pinot (SIGMOD 2021 paper)

## Signature ideas
- **H3, the hexagonal hierarchical spatial index.** Earth is projected onto an icosahedron
  (gnomonic projection per face) and tiled with hexagons at 16 resolutions; each finer cell is
  ~1/7 the area of its parent, and child IDs truncate to give ancestors. Why hexagons: a hexagon
  has exactly one neighbor distance (squares have two, triangles three), which simplifies
  gradient/smoothing analysis, and hexes minimize quantization error for objects in motion. You
  can't tile a sphere purely with hexagons, so 12 pentagons sit at icosahedron vertices — H3 uses
  Buckminster Fuller's orientation so all 12 land in oceans. 122 base cells cover the globe.
- **ETA-based matching, not nearest-distance (DISCO).** The geospatial index only produces
  candidates; the real ranking is done by computing road-network ETAs, and DISCO can match a
  rider with a driver who is currently on a trip but about to end nearby. Matt Ranney described
  the problem as a rolling traveling-salesman-like optimization across the city.
- **Ringpop: application-layer sharding via gossip.** Instead of putting hot driver/trip state
  behind a database, dispatch nodes gossip membership (SWIM protocol over TCP) and divide work on
  a consistent hash ring (FarmHash, replica points, red-black tree ring). Any node can accept any
  request and transparently forward it to the owner ("handle-or-forward"). It's an AP system —
  availability over consistency — that had scaled to thousands of nodes by 2015.
- **The driver phone as datastore of last resort.** For datacenter failover, dispatch
  periodically ships an encrypted state digest to each driver phone; if the datacenter dies, the
  phones upload their digests to the surviving DC to reconstruct in-flight trips. The fleet of
  phones is effectively an external replicated store.
- **Schemaless → Docstore.** In 2014 trip growth was outrunning Postgres, so Uber built
  Schemaless: an append-only, sharded MySQL datastore whose smallest unit is an immutable "cell."
  Its restrictive model made it hard to use as a general database (a Cassandra detour proved
  operationally immature at Uber scale), so it evolved into Docstore: real tables with schemas,
  MySQL ACID transactions, Raft-replicated 3–5 node partitions, strict serializability per
  partition, materialized views — a stateless query layer over a partitioned storage layer.
- **DOMA: taming 2,200 microservices.** Microservice sprawl meant debugging one issue could span
  ~50 services across 12 teams. Domain-Oriented Microservice Architecture groups services into
  ~70 domains behind gateways, arranged in 5 layers (infrastructure, business, product,
  presentation, edge) that constrain dependency direction and blast radius, with logic/data
  "extensions" so other teams can plug in without modifying core services.
- **Surge pricing on the grid.** Supply and demand are measured per hexagon; when demand outstrips
  available drivers in a cell, a multiplier raises the upfront price there, both rationing demand
  and pulling drivers toward the hot cells — the marketplace's feedback controller, computed on
  the same hex grid as everything else.

## Key numbers
- ~1M location writes/sec design target; driver phones update every 4 seconds; reads "many
  multiples higher" (2015) — https://highscalability.com/how-uber-scales-their-real-time-market-platform/
- Ringpop scaled to thousands of nodes; TChannel ~20x faster than the HTTP stack they replaced
  (2015, same talk) — https://highscalability.com/how-uber-scales-their-real-time-market-platform/
- H3: 16 resolutions, 122 base cells, 12 pentagons; compacting California from 10,633 res-6 hexes
  to 901 mixed-resolution cells (2018) — https://www.uber.com/blog/h3/
- ~2,200 critical microservices grouped into ~70 domains; integration work cut from 3 days to 3
  hours on one platform; platform support costs down an order of magnitude (2020) —
  https://www.uber.com/blog/microservice-architecture/
- Docstore: tens of millions of QPS at 99.99%+ availability over tens of petabytes (2021) —
  https://www.uber.com/blog/schemaless-sql-database/
- Fulfillment platform: >1M concurrent users, billions of trips/year across 10,000+ cities,
  billions of DB transactions/day on Spanner (2021) —
  https://www.uber.com/blog/fulfillment-platform-rearchitecture/
- Petabytes of data collected per day into the real-time stack (2021, SIGMOD paper) —
  https://arxiv.org/abs/2104.00087
- 3.1B trips in 2024, ~33M trips/day in Q4 2024 (2025 press release) —
  https://www.sec.gov/Archives/edgar/data/1543151/000154315125000004/uberq424earningspressrelea.htm

## Evolution timeline
- **2010–2013** — Outsourced MVP, then a Python monolith ("API") plus a Node.js dispatch
  monolith; Postgres for storage.
- **2014** — Trip growth threatens to exhaust Postgres; Schemaless (sharded MySQL) built
  (blogged 2016).
- **2015** — Dispatch rewritten as DISCO; Ringpop and TChannel open-sourced; S2 cells for
  geo-indexing; aggressive microservice adoption (~1,000+ services and ~2,000 engineers by 2016
  per HighScalability).
- **2018** — H3 hexagonal grid open-sourced; becomes the standard marketplace/pricing grid.
- **2020** — DOMA published: 2,200 services reorganized into ~70 domains and 5 layers.
- **2021** — Schemaless evolves into Docstore (Raft + MySQL, transactions); ground-up fulfillment
  re-architecture onto Google Cloud Spanner; real-time data infra (Kafka/Flink/Pinot) described
  at SIGMOD.

## Visualization hooks
1. **The hex zoom ladder.** A city under an H3 grid; zooming subdivides each hex into ~7 children
   (16 resolutions). Great for showing "same data structure from continent to street corner."
2. **Why hexagons.** Side-by-side square vs. hex grid with neighbor arrows: squares have 2
   distinct neighbor distances (edge vs. diagonal), hexes exactly 1 — then a moving dot showing
   lower quantization jitter on hexes.
3. **Unfolding the icosahedron.** Sphere → 20-face icosahedron → flattened net with 122 base
   cells, the 12 pentagons highlighted sitting safely in oceans.
4. **Dispatch in one shot.** Rider pin drops; a k-ring of hexes lights up around it; candidate
   driver dots get ETA labels via wiggly road paths (not straight lines); lowest ETA gets the
   offer; a declined offer cascades to the next.
5. **Surge as a heat map.** Hexes coloring from cool to hot as demand dots pile up faster than
   driver dots; multiplier numbers appear per cell; drivers visibly drift toward hot cells and
   the heat dissipates — a feedback loop on a map.
6. **The Ringpop ring.** Dispatch nodes on a hash ring exchanging gossip pulses; kill one node
   and watch its arc get re-owned by neighbors while requests "handle-or-forward" around.
7. **Phones as the backup datacenter.** A datacenter icon dies; thousands of tiny phones each
   hold a puzzle-piece "state digest" and stream them into the surviving DC, reassembling
   in-progress trips.
8. **Sprawl → domains.** 2,200 tangled service dots resolving into ~70 labeled domain clusters
   stacked in 5 layers with one-way dependency arrows — the DOMA cleanup as a graph untangling.

## Sources
- "H3: Uber's Hexagonal Hierarchical Spatial Index" — Uber Engineering Blog, 2018.
  https://www.uber.com/blog/h3/ — Why hexagons, projection, resolutions, pentagons, use cases.
  Primary.
- H3 documentation — h3geo.org (open-source project docs). https://h3geo.org/docs/ — API
  (geoToH3, kRing, compact), cell statistics. Primary.
- "How Uber Scales Their Real-time Market Platform" — HighScalability, 2015.
  https://highscalability.com/how-uber-scales-their-real-time-market-platform/ — Detailed notes
  on Matt Ranney's talk: DISCO, supply/demand services, S2, Ringpop, TChannel, failover.
  Secondary (faithful writeup of a primary talk).
- "Scaling Uber's Real-time Market Platform" — Matt Ranney, QCon London 2015, via InfoQ.
  https://www.infoq.com/presentations/uber-market-platform/ — The primary talk itself (video).
  Primary.
- "Uber Engineering's Ringpop" — Uber Engineering Blog, 2015.
  https://www.uber.com/blog/ringpop-open-source-nodejs-library/ — SWIM gossip, consistent
  hashing, handle-or-forward. Primary. (Deep dive: https://ringpop.readthedocs.io/en/latest/architecture_design.html)
- "Designing Schemaless, Uber Engineering's Scalable Datastore Using MySQL" — Uber Engineering
  Blog, 2016. https://www.uber.com/blog/schemaless-part-one-mysql-datastore/ — The 2014 Postgres
  crisis, cells, append-only design. Primary.
- "Evolving Schemaless into a Distributed SQL Database" (Docstore) — Uber Engineering Blog, 2021.
  https://www.uber.com/blog/schemaless-sql-database/ — Docstore architecture, Raft, scale.
  Primary.
- "Introducing Domain-Oriented Microservice Architecture" — Uber Engineering Blog, 2020.
  https://www.uber.com/blog/microservice-architecture/ — DOMA concepts and results. Primary.
- "Uber's Fulfillment Platform: Ground-up Re-architecture" — Uber Engineering Blog, 2021.
  https://www.uber.com/blog/fulfillment-platform-rearchitecture/ — Spanner move, scale figures.
  Primary. (Companion: https://www.uber.com/blog/building-ubers-fulfillment-platform/)
- "Real-time Data Infrastructure at Uber" — Fu & Soman, SIGMOD 2021.
  https://arxiv.org/abs/2104.00087 — Kafka/Flink/Pinot stack, PBs/day. Primary (peer-reviewed).
- "Uber Marketplace: Surge pricing" — Uber (product/marketplace page, current).
  https://www.uber.com/us/en/marketplace/pricing/surge-pricing/ — Official high-level surge
  mechanics. Primary but non-technical.
- Uber Q4 2024 earnings press release — SEC filing, 2025.
  https://www.sec.gov/Archives/edgar/data/1543151/000154315125000004/uberq424earningspressrelea.htm
  — Trip volume figures. Primary (financial disclosure).
- "Lessons Learned from Scaling Uber to 2000 Engineers, 1000 Services..." — HighScalability, 2016.
  https://highscalability.com/lessons-learned-from-scaling-uber-to-2000-engineers-1000-ser/ —
  Microservice-era growth context. Secondary.
