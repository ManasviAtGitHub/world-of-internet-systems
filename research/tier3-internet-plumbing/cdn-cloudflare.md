# CDNs and Cloudflare — Moving the Internet Next Door

## One-line hook
Cloudflare answers for more than 20% of the web from 330+ cities using one weird trick — announcing the same IP addresses everywhere at once — so your request, a 31 Tbps DDoS, and a serverless function all land at a data center a few milliseconds away.

## The core problem
Physics: an origin server in Virginia is ~250 ms round-trip from Sydney, and every round trip (TCP, TLS, request) pays it again. Economics: one origin cannot absorb a planet's worth of traffic — legitimate or malicious. CDNs solve both with the same move: put servers close to every user, terminate connections there, serve cached content there, and only bother the origin when something is genuinely new. Cloudflare's twist was doing this as a reverse proxy for *everything* (not just static files) — DNS, TLS, security, and eventually compute — turning "a cache in front of your site" into a programmable layer of the internet.

## How it works

**Getting the user to the nearest edge — two schools:**
- **DNS-based steering (classic Akamai model):** every user gets a *different* IP for the same hostname; the CDN's DNS looks at where the resolver is and hands back a nearby cluster's address. Fine-grained control, but depends on resolver location and TTL churn.
- **Anycast (Cloudflare's model):** every data center announces the *same* IP prefixes via BGP; the internet's own routing delivers each packet to the topologically nearest site. No per-user decisions at all — BGP is the load balancer. Cloudflare's early primer measured ~9.5 ms to the nearest edge vs ~50 ms to the unicast origin. Bonus: a DDoS aimed at one IP is automatically sprayed across every data center on Earth — "increase the surface area to absorb" the attack.

**The request path (cache hierarchy):**
1. User's TCP/QUIC + TLS handshakes terminate at the nearest edge (~10-30 ms RTT instead of 100+). The edge holds the certificate, speaks TLS 1.3/HTTP/3 to the user regardless of what the origin supports, and re-encrypts on a separate connection to the origin.
2. **Edge cache (lower tier):** if the object is cached locally — hit, done, single-digit-ms. Cloudflare's earliest architecture already answered ~75% of requests at the edge without touching the origin.
3. **Shield / upper tier:** on a miss, tiered cache sends the request not to the origin but to a designated larger data center; only upper tiers may contact origins, and they fan results back down. Misses from 300 sites concentrate into a few, so the origin sees a handful of requests instead of hundreds, and inter-Cloudflare links are faster than the public path to origin.
4. **Origin:** only truly-uncacheable or expired content gets through. Static assets follow Cache-Control/TTL rules; modern setups cache HTML too, and Workers can generate responses without any origin.

**Absorbing a DDoS:** attack packets follow the same anycast routes as users, so a 31.4 Tbps flood arrives pre-fragmented across hundreds of sites, each handling a slice within local capacity (total network capacity 500 Tbps — attack peaks are a fraction of headroom). Fingerprinting and dropping happens in kernel/XDP at every edge, autonomously — Cloudflare reports mitigating thousands of attacks daily with no human, including the record 31.4 Tbps burst (35 seconds long) and 5,376 attacks auto-mitigated *per hour* on average in 2025.

**Edge compute (Workers):** instead of spinning containers, Cloudflare runs customer code as V8 isolates — the same sandbox Chrome uses for tabs — inside one process. Cold starts are ~5 ms (vs 500 ms-10 s for container serverless) and a worker's overhead is ~3 MB vs ~35 MB for a Node Lambda, cheap enough that *every* server in *every* data center runs the platform; your code is simply everywhere.

Component list (plain text):
- Anycast IP prefixes announced from 330+ cities via BGP (AS13335)
- Edge data centers: TLS termination, HTTP/3, cache, WAF, DDoS filters, Workers runtime
- Tiered cache: lower-tier edges → upper-tier shields → customer origin
- Authoritative DNS + 1.1.1.1 resolver on the same network
- Quicksilver: global key-value store pushing config/rules to every edge in seconds
- Workers platform: V8 isolates, KV, Durable Objects, R2 on every server

## Signature ideas

- **Anycast outsources load balancing to the internet itself.** One IP, hundreds of locations, zero steering logic — BGP's "nearest exit" behavior does user routing, failover (withdraw a sick site's announcement and traffic re-flows), and DDoS dispersion as free side effects. The elegant part: clients can't tell any of this is happening.
- **Terminate early, and the speed of light matters less.** Handshakes are per-RTT costs; moving the TLS endpoint from 150 ms away to 15 ms away makes the whole connection setup ~10x faster before a single byte of content moves. The origin's distance then only matters on cache misses — which tiered caching makes rare.
- **The cache is a funnel, not a mirror.** Edge → shield → origin means each layer sees an order of magnitude fewer misses than the one below. The origin of a heavily-trafficked site can be a single modest server because the hierarchy eats the fan-in.
- **Capacity is the DDoS strategy.** Cloudflare's stated design: peak legitimate utilization is a small fraction of the 500 Tbps interconnect, and the rest is deliberately idle armor. Against volumetric attacks there is no cleverness that substitutes for headroom-times-dispersion.
- **Isolates made "run everywhere" economical.** Containers are too heavy to run every customer's code in 330 cities; 3 MB isolates with 5 ms cold starts are light enough that deployment location stops being a choice. This is the architectural unlock behind "region: Earth."
- **Shared fate cuts both ways.** Fronting 20%+ of the web with one network means one bad config is a mini internet outage (July 2019: one regex, 27 minutes, 80% of traffic dropped). Centralizing the edge trades many small failures for rare, spectacular, front-page ones.

## Key numbers
- 500 Tbps of interconnection capacity; 330+ cities in 125+ countries; 13,000+ directly connected networks (Cloudflare "500 Tbps" post, Apr 2026; https://blog.cloudflare.com/500-tbps-of-capacity/)
- Protects/serves "more than 20% of the web" (Cloudflare, 2026; same URL)
- Record DDoS mitigated: 31.4 Tbps, lasting ~35 seconds, blocked autonomously (Cloudflare Q4 2025 DDoS report, Feb 2026; https://blog.cloudflare.com/ddos-threat-report-2025-q4/)
- Escalation trail: 7.3 Tbps (May 2025) → 11.5 Tbps (Sept 2025) → 31.4 Tbps (Dec 2025) (Cloudflare, 2025-26; https://blog.cloudflare.com/defending-the-internet-how-cloudflare-blocked-a-monumental-7-3-tbps-ddos/)
- 5,376 DDoS attacks auto-mitigated per hour on average in 2025; +121% YoY (Cloudflare Q4 2025 report, 2026; https://blog.cloudflare.com/ddos-threat-report-2025-q4/)
- Workers isolates: ~5 ms cold start vs 500 ms-10 s for container serverless; ~3 MB vs ~35 MB memory per tenant (Cloudflare, 2018; https://blog.cloudflare.com/cloud-computing-without-containers/)
- Early anycast edge: ~9.5 ms vs ~50 ms to origin; ~75% of requests answered at edge (Cloudflare anycast primer, 2011; https://blog.cloudflare.com/a-brief-anycast-primer/)
- July 2019 outage: 27 minutes, ~80% traffic drop, CPUs pinned at 100% globally (Cloudflare postmortem, 2019; https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/)
- ~23% of all websites proxied by Cloudflare; ~95% of the internet-connected population within 50 ms of an edge (Cloudflare-derived stats roundups, 2026 — verify against cloudflare.com/network/; https://www.demandsage.com/cloudflare-statistics/)

## Famous incidents
- **The regex that stopped 20% of the web — July 2, 2019.** A new WAF rule contained `.*(?:.*=.*)` — catastrophic backtracking made the regex engine's work explode, pinning every HTTP-serving CPU on every Cloudflare server to 100%. Global 502s, ~80% of traffic dropped. WAF rules bypassed staged rollout *by design* (they exist to counter emergent threats fast) and deployed worldwide in seconds via Quicksilver. The kill switch took 27 minutes partly because the engineers' own auth (Cloudflare Access) ran on the broken edge. Fixes: CPU limits restored, all 3,868 WAF rules audited, migration to linear-time regex engines (re2/Rust), staged rollouts even for emergencies. Teaches: global config pipelines are the sharpest knife in the drawer; O(2^n) is an outage class; don't lock your fire extinguisher inside the burning building.
- **Absorbing the record floods — 2025.** The 31.4 Tbps December 2025 attack (and the 7.3 and 11.5 Tbps records before it) lasted seconds and was dispersed across the anycast network and dropped by autonomous systems with no human paged. The story is the *non-event*: customers mostly learned about it from the blog post. Teaches: anycast + headroom converts an apocalypse into a graph annotation.
- **Facebook Oct 2021, seen from the edge.** Not Cloudflare's outage — but 1.1.1.1 absorbing a 30x retry storm while keeping <10 ms answers is the CDN-scale counterpart to the BGP story, and shows how one company's outage becomes load on everyone else's infrastructure. (https://blog.cloudflare.com/october-2021-facebook-outage/)
- **Dyn 2016 as the pre-CDN cautionary tale.** A single authoritative-DNS provider under Mirai attack took down Twitter, Netflix, GitHub for hours — the incident that pushed the industry toward anycast-everywhere DNS and multi-provider setups (see dns.md). Teaches: the edge model exists because concentrated chokepoints fail loudly.

## Visualization hooks
- **Unicast vs anycast split-screen world map**: same IP; left, all arrows converge on Virginia; right, arrows snap to nearest of 330 dots. Then replay with attack traffic — left melts, right disperses.
- **The RTT tax meter**: a Sydney user's handshake to Virginia (3 × 250 ms ticks) vs to a local edge (3 × 15 ms) — animate the meter running.
- **Cache funnel**: 1M requests raining on 300 edges → ~250k misses converge on a few shields → a trickle of thousands reaches one small origin server, dozing.
- **DDoS absorption as rainfall**: 31.4 Tbps drawn as a storm cloud over the whole map, each data center's gutter draining its share; a bar shows attack peak vs 500 Tbps capacity.
- **The regex bomb**: a tiny string `x=x` entering the pattern `.*.*=.*` and exploding into an exponentially-branching backtracking tree while a CPU gauge sweeps to 100% — then the same input through a linear-time engine as a straight line.
- **Config at the speed of Quicksilver**: one rule pushed from HQ lighting up 330 cities in seconds — first as the hero (threat response), then the same animation as the villain (July 2019).
- **Containers vs isolates**: an apartment building (one process) with hundreds of tiny rooms (isolates, 3 MB, 5 ms move-in) vs a suburb of whole houses (containers, 35 MB, 500 ms+) — why only one fits in 330 cities.
- **TLS termination handoff**: the encrypted tunnel from the user visibly ending at the edge (certificate badge), a second tunnel continuing to origin — two padlocks, one short and fast, one long and rare.

## Sources
- "500 Tbps of capacity: 16 years of scaling our global network" — Cloudflare blog, Apr 2026. Current capacity, cities, peers, >20%-of-web claim, capacity-as-DDoS-buffer doctrine. Primary. https://blog.cloudflare.com/500-tbps-of-capacity/
- "Details of the Cloudflare outage on July 2, 2019" — Cloudflare blog (jgc), 2019. Full postmortem: regex, timeline, 11 root causes, remediations. Primary; one of the best postmortems ever published. https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/
- "A Brief Primer on Anycast" — Cloudflare blog, 2011. Anycast vs unicast, latency measurements, DDoS surface-area argument. Primary (early numbers; network is ~100x bigger now). https://blog.cloudflare.com/a-brief-anycast-primer/
- "Cloud Computing without Containers" — Cloudflare blog (Zack Bloom), 2018. Isolate architecture, 5 ms cold start, 3 MB overhead, economics. Primary. https://blog.cloudflare.com/cloud-computing-without-containers/
- "Introducing: Smarter Tiered Cache Topology" — Cloudflare blog, 2021. Lower/upper tier design, only-shields-contact-origin rule. Primary. https://blog.cloudflare.com/introducing-smarter-tiered-cache-topology-generation/
- "2025 Q4 DDoS threat report" — Cloudflare blog, Feb 2026. 31.4 Tbps record, 5,376 attacks/hour, yearly trends. Primary measurement. https://blog.cloudflare.com/ddos-threat-report-2025-q4/
- "How Cloudflare blocked a monumental 7.3 Tbps DDoS" — Cloudflare blog, Jun 2025. Anatomy of a hyper-volumetric attack and autonomous mitigation. Primary. https://blog.cloudflare.com/defending-the-internet-how-cloudflare-blocked-a-monumental-7-3-tbps-ddos/
- Cloudflare statistics roundup — DemandSage, 2026. Aggregated %-of-websites and 50 ms-population figures. Secondary; verify key numbers against cloudflare.com/network/ before publishing. https://www.demandsage.com/cloudflare-statistics/
