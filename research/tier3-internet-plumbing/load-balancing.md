# Load Balancing

## One-line hook
Every request you send to Google traverses at least four different load balancers before it touches an application server — and the cleverest one picks between just two random options.

## The core problem
One server can only handle so many connections, and any server can die at any moment.
Load balancing solves both problems at once:
- It makes a fleet of machines look like one endpoint (a single IP or hostname).
- It spreads incoming traffic so no machine drowns.
- It quietly routes around machines that fail, ideally before users notice.

The hard parts hide in the details:
- How do you spread load *evenly* when your information about server load is always slightly stale?
- How do you keep a TCP connection glued to the same backend when the set of backends changes underneath it?
- And how do you load-balance the load balancers themselves?

## How it works
Modern internet traffic passes through a layered funnel, each layer balancing at a different granularity:

1. **DNS / global load balancing.** A user resolves `example.com`. Either DNS returns different IPs per region (GeoDNS), or — the Cloudflare approach — every data center announces the *same* IP via BGP anycast, and internet routing itself delivers the packet to the topologically nearest site. Cloudflare described running its whole edge this way as early as 2013, with all (then 23) data centers announcing identical IP blocks.
2. **ECMP at the router.** Inside a data center, several L4 load balancer machines all announce the service's virtual IP (VIP) over BGP. The router uses Equal-Cost Multi-Path (ECMP) to hash each packet's 5-tuple and spray flows evenly across the L4 tier. This is how Google's Maglev and GitHub's GLB scale a single IP across many balancer machines.
3. **L4 load balancing (transport layer).** The L4 balancer sees only packet headers: source/destination IP, port, protocol. It does not parse HTTP and does not terminate TLS. It picks a backend — consistent hashing plus a local connection table so every packet of a flow lands on the same backend — and forwards the packet, often by GRE encapsulation so the backend can reply directly to the client (direct server return, keeping the bulky response traffic off the balancer). Cheap per packet, enormous throughput.
4. **L7 load balancing (application layer).** A reverse proxy (NGINX, HAProxy, Envoy) terminates TLS, parses the HTTP request, and can:
   - route on path, headers, or cookies (`/api` to one pool, `/static` to a CDN pool);
   - retry a failed request against a different backend (invisible to the user);
   - enforce timeouts, rate limits, and circuit breaking;
   - keep sticky sessions via cookies when state demands it.
   Smarter than L4, but costs far more CPU per request — which is why it sits *behind* the L4 tier, scaled horizontally.
5. **Health checking wraps every layer.** Active checks (periodic HTTP/TCP probes with fail/rise thresholds) plus passive detection (watching real traffic for errors and timeouts — Envoy calls this "outlier detection" and temporarily ejects misbehaving hosts). Two courtesies keep changes smooth: *slow start* ramps traffic to recovering hosts gradually, and *connection draining* lets departing hosts finish in-flight requests.

Plain-text component list:
- Client → DNS resolver (GeoDNS answer or anycast IP)
- BGP anycast → nearest data center
- Router ECMP → one of N L4 balancer machines
- L4 balancer: VIP table, consistent-hash lookup table, connection tracking table, encapsulator
- L7 proxy: TLS termination, HTTP router, retry/timeout logic, balancing algorithm
- Backend pool + health checker (active probes + passive outlier detection)
- Control plane: config distribution, BGP announcements, backend registration

Concrete end-to-end flow (Google-style):
1. Your packet to a Google VIP hits a router at the edge of a Google data center.
2. The router ECMP-hashes the 5-tuple and forwards to one of many Maglev machines.
3. Maglev looks up the VIP, checks its connection table for an existing flow, otherwise consistent-hashes the 5-tuple into its lookup table to pick a service endpoint.
4. Maglev GRE-encapsulates the packet to that endpoint.
5. The endpoint decapsulates, processes, and replies *straight to you* — the return path bypasses Maglev entirely.
6. If a Maglev machine dies, ECMP re-spreads its flows to siblings, and because every Maglev computes the identical hash table, the siblings send those flows to the same backends. Connections survive.

## Signature ideas
- **L4 vs L7 is about what the balancer can see.** An L4 device sees IP/port tuples and moves packets; an L7 proxy sees full requests and moves *requests*. That's why L4 balancers reach line rate on commodity hardware while L7 proxies buy routing intelligence at CPU cost. Real stacks chain them: L4 in front, L7 behind.
- **Round robin and least-connections are the folk algorithms.** Round robin deals requests like cards — fine when requests are uniform, unfair when some are 100x heavier. Least-connections uses live connection count as a proxy for load — better, but it assumes the balancer sees all the traffic, which stops being true the moment you run many independent balancers.
- **Power of two choices (P2C).** Picking the less-loaded of *two randomly chosen* servers captures almost all the benefit of scanning every server: Mitzenmacher's 1996 thesis showed the expected max load drops exponentially (from ~log n/log log n for pure random placement to ~log log n with two choices). Better yet, it avoids the "herd" failure mode of pick-the-best: with stale load data, every balancer stampedes the same apparently-idle server, crushes it, then stampedes the next one. Marc Brooker's simulations (2012) show best-of-2 beats best-overall precisely when load information is delayed. NGINX shipped it as "Random with Two Choices" in 2018; Envoy uses P2C as the default for least-request.
- **Consistent hashing: the ring.** Karger et al. (1997) hash both servers and keys onto a circle; each key belongs to the next server clockwise. Adding or removing one server remaps only ~1/N of keys instead of nearly all of them (the naive mod-N disaster). Amazon's Dynamo (2007) added *virtual nodes* — each physical server appears at many points on the ring — to smooth statistical load imbalance and let heterogeneous servers take proportional shares.
- **Maglev hashing: trade a little disruption for perfect balance.** Ring hashing balances only approximately. Maglev instead builds a prime-sized lookup table (65,537 entries in the paper's tests) where each backend fills slots via its own permutation, taking turns — guaranteeing near-perfectly equal shares, at the cost of slightly more remapping on membership change than a ring. Every Maglev machine computes the identical table independently, with no coordination — so any Maglev can handle any packet of any flow.
- **Connection tracking papers over hash changes.** Consistent hashing decides where *new* flows go; a local connection table remembers where *existing* flows went, so backend churn doesn't break established TCP connections. GitHub's GLB goes further: rendezvous hashing yields a primary/secondary server pair per flow, and the secondary can complete a handshake the primary started, so even director churn is survivable.
- **Health checks are a distributed-systems problem in miniature.** A backend isn't "up" or "down" — it's "some checkers can currently reach it." Systems hedge with thresholds (eject after k consecutive failures, readmit after m successes), blend active probes with passive signals from live traffic, and treat a mass-ejection event ("everything looks down") as evidence the *checker* is partitioned, not the fleet.

## Key numbers
- A single Maglev machine saturates a 10 Gbps link with minimum-sized packets; Maglev has served Google traffic since 2008 (paper: 2016) — https://www.usenix.org/sites/default/files/nsdi16-paper-eisenbud-update.pdf
- Two random choices reduce expected max load from Θ(log n/log log n) to Θ(log log n) — an exponential improvement (Mitzenmacher, 1996) — https://www.eecs.harvard.edu/~michaelm/postscripts/mythesis.pdf
- Best-of-2 outperforms "pick the global best" once load information is stale — simulation writeup (2012) — https://brooker.co.za/blog/2012/01/17/two-random.html
- NGINX shipped "Random with Two Choices" in NGINX Open Source 1.15.1 / NGINX Plus R16 (2018) — https://www.f5.com/company/blog/nginx/nginx-power-of-two-choices-load-balancing-algorithm
- Cloudflare's Unimog L4 balancer runs on *every* edge server and costs under 1% of CPU, using XDP in the Linux kernel (2020) — https://blog.cloudflare.com/unimog-cloudflares-edge-load-balancer/
- Cloudflare was doing global load balancing purely with anycast — all (then 23) data centers announcing the same IPs — in 2013 — https://blog.cloudflare.com/cloudflares-architecture-eliminating-single-p/
- GitHub open-sourced GLB Director, its ECMP + rendezvous-hashing L4 tier, in August 2018 — https://github.blog/engineering/infrastructure/glb-director-open-source-load-balancer/
- Consistent hashing was published in 1997 (Karger et al., STOC) — originally for web caching, a decade before Dynamo made it famous — https://dl.acm.org/doi/10.1145/258533.258660

## Who uses it and how
- **Google (Maglev):** every packet to a Google service or Google Cloud network load balancer since 2008 passes through Maglev — commodity Linux servers replacing hardware appliances, scaled horizontally behind router ECMP, with Maglev hashing ensuring any machine can serve any flow (NSDI 2016 paper).
- **Cloudflare (anycast + Unimog):** no dedicated load balancer boxes at all — every edge server in every data center runs the full stack; anycast delivers packets to the nearest city, and Unimog (XDP, in-kernel) rebalances flows between servers within a data center at <1% CPU cost (2020).
- **GitHub (GLB):** a single IP for github.com scaled across many machines via BGP + ECMP + a DPDK-based director using rendezvous hashing, designed so directors keep no shared connection state yet don't break TCP flows during membership changes (2018).
- **Amazon (Dynamo lineage):** Dynamo's 2007 paper made consistent hashing with virtual nodes the standard pattern for partitioning both storage and traffic; the same ring idea now routes requests in caches (memcached/Ketama), CDNs, and databases (Cassandra, DynamoDB).
- **Lyft and the service-mesh world (Envoy):** Envoy, built at Lyft and now the data plane of most service meshes, does L7 balancing with P2C least-request by default plus passive outlier detection — load balancing migrated from a central tier into a sidecar next to every service.

## Visualization hooks
- **The funnel:** one request descending through DNS → anycast world map → router ECMP fan-out → L4 tier → L7 proxies → backend pool; camera dolly from planet scale to rack scale.
- **Consistent hashing ring, live:** keys as dots orbiting to the next clockwise node; kill a node and watch only its arc of keys slide to the neighbor while the rest stay put; contrast with mod-N where nearly every dot jumps at once.
- **Virtual nodes:** a ring with 3 fat arcs (lumpy, unfair) exploding into 300 thin interleaved slivers; a per-server load bar chart flattens as virtual node count rises.
- **Balls into bins race:** split screen — pure random dropping balls into bins vs P2C peeking at two bins before dropping; watch the tallest stack diverge (log n/log log n vs log log n).
- **The herd:** many independent balancers all reading the same stale "server 7 is idle" report and simultaneously flooding server 7, which reddens and dies; then P2C's randomness dissolving the stampede into an even shimmer.
- **Maglev table fill:** backends taking turns claiming slots of a 65,537-entry table via their permutation sequences — like players alternating draft picks — ending in a perfectly striped table; delete a backend and watch how few slots change hands.
- **Health-check state machine:** a backend cycling healthy → suspect (2 failed probes) → ejected → slow-start ramp back to full traffic, drawn as a traffic-light timeline against the live request stream.
- **Anycast:** the same IP address glowing in hundreds of cities; a packet from each continent flowing to its nearest glow; one city goes dark and its traffic refracts to neighbors automatically — no failover logic anywhere, just routing.

## Sources
- "Maglev: A Fast and Reliable Software Network Load Balancer" — Eisenbud et al., Google, NSDI 2016. The L4 architecture, Maglev hashing, ECMP scaling, 10Gbps result, serving Google since 2008. (Primary; peer-reviewed paper.) https://www.usenix.org/sites/default/files/nsdi16-paper-eisenbud-update.pdf
- "The Power of Two Choices in Randomized Load Balancing" — Michael Mitzenmacher, PhD thesis, 1996. The original P2C result and its analysis. (Primary; academic.) https://www.eecs.harvard.edu/~michaelm/postscripts/mythesis.pdf
- "The power of two random choices" — Marc Brooker (AWS), blog, 2012. Simulations showing why best-of-2 beats best-overall under stale information; herd behavior. (Secondary but authoritative practitioner analysis.) https://brooker.co.za/blog/2012/01/17/two-random.html
- "NGINX and the 'Power of Two Choices' Load-Balancing Algorithm" — Owen Garrett, NGINX/F5 blog, 2018. Random-with-two-choices in NGINX and when it matters (fleets of independent balancers). (Primary vendor engineering post.) https://www.f5.com/company/blog/nginx/nginx-power-of-two-choices-load-balancing-algorithm
- "Test driving 'power of two random choices' load balancing" — HAProxy blog, 2018. Empirical comparison of P2C vs least-connections in HAProxy. (Primary vendor engineering post.) https://www.haproxy.com/blog/power-of-two-load-balancing
- "Unimog — Cloudflare's edge load balancer" — Cloudflare blog, 2020. XDP-based L4 balancing on every edge server, <1% CPU, interplay with anycast and DDoS mitigation. (Primary engineering post.) https://blog.cloudflare.com/unimog-cloudflares-edge-load-balancer/
- "Load Balancing without Load Balancers" — Cloudflare blog, 2013. Global anycast + DNS design; all data centers announcing the same IPs. (Primary engineering post.) https://blog.cloudflare.com/cloudflares-architecture-eliminating-single-p/
- "GLB: GitHub's open source load balancer" — GitHub Engineering blog, 2018. ECMP + rendezvous hashing, primary/secondary flow handoff, stateless directors. (Primary engineering post.) https://github.blog/engineering/infrastructure/glb-director-open-source-load-balancer/
- "Consistent Hashing and Random Trees" — Karger et al., STOC 1997. The original ring construction. (Primary; peer-reviewed paper.) https://dl.acm.org/doi/10.1145/258533.258660
- "Dynamo: Amazon's Highly Available Key-value Store" — DeCandia et al., SOSP 2007. Virtual nodes on the consistent-hashing ring; partitioning in practice. (Primary; peer-reviewed paper.) https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
- Envoy proxy documentation — load balancing algorithms (P2C least-request default) and outlier detection. (Primary documentation.) https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing
- "What is Anycast?" — Cloudflare Learning Center. Plain-language anycast explainer for the global layer. (Secondary; vendor educational page.) https://www.cloudflare.com/learning/cdn/glossary/anycast-network/
