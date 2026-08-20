# BGP — The Protocol That Holds the Internet Together with Trust

## One-line hook
The entire internet's map is maintained by ~78,000 networks shouting routes at each other and mostly believing whatever they hear — which is how one bad command deleted Facebook from the internet for six hours, and one Pakistani ISP once accidentally became YouTube for the whole world.

## The core problem
The internet isn't one network — it's roughly 78,000 independently-run networks ("autonomous systems," ASes: ISPs, clouds, universities, Facebook) that must somehow agree on how to reach every one of ~1.05 million address blocks, with no central authority, while each operator keeps its business policies (who pays whom for transit) private. BGP (Border Gateway Protocol, RFC 4271) is the gossip protocol that solves this: each AS announces "these prefixes live here, via this path of ASes," neighbors propagate what they hear, and every router assembles its own view of the world. It prioritizes autonomy and policy over correctness and security — a 1989 design trade the internet still lives with.

## How it works

- **Autonomous systems.** Each network gets an AS number (Facebook = AS32934, Cloudflare = AS13335). At the end of 2025 there were ~77,900 ASes visible in the IPv4 table — ~11,400 transit networks that carry others' traffic and ~66,500 "stub" edge networks (APNIC).
- **Announcements and path-vector routing.** An AS originates a prefix ("129.134.30.0/23 is mine") to its BGP neighbors over TCP sessions. Each neighbor prepends its own AS number and re-announces, so a route carries its full AS path, e.g. `[3356, 32934]`. Loops are detected by seeing your own ASN in the path. This is "path-vector": like distance-vector but the metric is the actual path, enabling both loop-freedom and policy filtering.
- **Route selection.** Among competing routes for a prefix, a router prefers (roughly): highest local preference (business policy — customer routes over peer routes over paid transit), then shortest AS path, then tiebreakers. Crucially, **longest prefix match beats everything**: a /24 route always wins over a covering /22, no matter the path. This is the mechanism hijackers exploit.
- **Propagation.** A new announcement ripples across the internet's ~78k ASes in seconds to minutes; withdrawal ripples the same way. There is no acknowledgment, no global view — only each router's local table (~1.05M IPv4 + ~242k IPv6 routes in 2025, per APNIC).
- **The trust model (or absence of one).** Classic BGP has zero built-in validation: any AS can announce any prefix, and neighbors will generally believe it, filtered only by whatever manual prefix-lists operators maintain. Hijacks happen because lying (or fat-fingering) is protocol-legal.
- **RPKI: retrofitting proof of ownership.** The Resource Public Key Infrastructure lets address holders publish signed Route Origin Authorizations (ROAs) — "AS32934 may originate 129.134.30.0/23" — anchored at the five regional registries (ARIN, RIPE, APNIC, LACNIC, AFRINIC). Routers fetch validated data via the RTR protocol and can drop "RPKI-invalid" announcements. Limitation: it validates only the *origin*, not the path — a hijacker can still forge the origin AS at the end of a fake path (Cloudflare RPKI post). Coverage hit a record ~67% of announced prefixes with valid ROAs by mid-2026, up from single digits in 2018.

**Narrative spine — Facebook, October 4, 2021:**
1. During routine maintenance, an engineer ran a command meant to *assess* global backbone capacity; it instead disconnected Facebook's entire backbone. An audit tool existed to catch exactly this, but a bug "prevented it from properly stopping the command" (Meta postmortem).
2. Facebook's DNS servers are designed to withdraw their own BGP advertisements if they can't reach the data centers (a health check meant to stop them serving stale answers). With the backbone down, *all* of them concluded they were unhealthy — and withdrew.
3. At ~15:40 UTC Cloudflare saw a spike of withdrawals from AS32934; by 15:58 the DNS prefixes (129.134.30.0/23, 185.89.218.0/23) were gone from the global table. Facebook's nameservers were up and running — but no route on Earth led to them. "As if someone had pulled the cables from their data centers all at once" (Cloudflare).
4. Cascade: DNS worldwide returned SERVFAIL for facebook.com/instagram/WhatsApp; apps retried furiously (30x query load on 1.1.1.1). Inside Meta, the same DNS/network powered badge readers, internal tools, and remote access — engineers reportedly couldn't get into buildings or dashboards, and had to physically deploy people to data centers where hardware is deliberately hard to modify.
5. Recovery required manual, on-site restarts and a careful staged power-up (data centers swing "tens of megawatts"). BGP announcements returned ~21:00 UTC; DNS recovered ~21:20. Roughly six hours offline — self-inflicted, no attacker involved.

Component list (plain text):
- ~78k autonomous systems (ASNs), from tier-1 transit carriers to stub networks
- eBGP sessions between ASes; iBGP inside them; all over TCP/179
- Global routing table: ~1.05M IPv4 + ~242k IPv6 prefixes (end of 2025)
- Route collectors/monitors: RIPE RIS, RouteViews, Cloudflare Radar
- RPKI: RIR trust anchors → signed ROAs → validators (Routinator etc.) → RTR feed → routers
- Internet exchange points (IXPs) where ASes physically peer

## Signature ideas

- **The internet is a federation, not a network.** BGP's real product is not routes but *sovereignty*: every AS applies its own commercial policy (customers > peers > transit) and reveals nothing about why. That's why the internet could grow without a ruler — and why nobody can simply "fix" it centrally.
- **Reachability is a broadcasted rumor.** Your servers being up is irrelevant; you exist on the internet only while other routers hold a current rumor of how to reach you. Facebook 2021 is the purest demonstration: healthy data centers, zero reachability.
- **Longest-prefix match is the hijacker's lever.** Announce a more specific slice of someone's space and every router prefers you automatically — no deception skills needed. Pakistan's /24 beat YouTube's /22 globally in minutes; YouTube clawed traffic back by announcing even-more-specific /25s. Attack and defense used the same rule.
- **Automated self-defense can become self-destruction.** Facebook's DNS "withdraw if unhealthy" logic is sane for one failing site; applied globally during a backbone outage it amputated the company from the internet. Classic correlated-failure lesson: the safety mechanism assumed failures would be local.
- **The recovery paradox.** Big outages disable the tools needed to fix them — Meta's engineers lost internal dashboards, remote access, even door badges to the very outage they were debugging. Out-of-band access is a design requirement, not a nicety.
- **RPKI is the internet retrofitting a deed registry.** Thirty years in, the routing system is finally getting cryptographic proof of "who owns this block" — but only origin validation, only ~2/3 coverage, and only if operators actually drop invalids. Security retrofits on federated systems move at the speed of the slowest incentive.

## Key numbers
- ~1,050,000 IPv4 prefixes in the global table at end of 2025 (+5% YoY); ~241,800 IPv6 (APNIC "BGP in 2025," Jan 2026; https://blog.apnic.net/2026/01/08/bgp-in-2025/)
- ~77,900 IPv4 ASNs (11,400 transit, 66,500 stub); ~36,100 announce IPv6 (APNIC, 2026; same URL)
- RPKI ROA coverage record: 67.43% of announced prefixes (1,065,730 of 1,580,470 routes) as of June 29, 2026 (ipregistry.co citing RPKI monitors, 2026; https://ipregistry.co/blog/rpki-blind-spots/; see also https://blog.cloudflare.com/rpki-updates-data/)
- Facebook outage: withdrawals spiked 15:40 UTC; DNS prefixes 129.134.30.0/23 and 185.89.218.0/23 withdrawn by 15:58; BGP back ~21:00, DNS ~21:20 — ~5.5-6 h (Cloudflare, 2021; https://blog.cloudflare.com/october-2021-facebook-outage/)
- 30x DNS query volume on 1.1.1.1 during that outage (Cloudflare, 2021; same URL)
- YouTube hijack: started 18:47 UTC Feb 24 2008, fully resolved 21:01 — ~2 h 14 min global impact (RIPE NCC RIS case study, 2008; https://www.ripe.net/publications/news/youtube-hijacking-a-ripe-ncc-ris-case-study/)
- 5 RIR trust anchors underpin all of RPKI (Cloudflare, 2018; https://blog.cloudflare.com/rpki/)

## Famous incidents
- **Pakistan hijacks YouTube — Feb 24, 2008.** Pakistan Telecom (AS17557), ordered to censor YouTube domestically, announced 208.65.153.0/24 — a more-specific of YouTube's 208.65.152.0/22. Upstream PCCW (AS3491) propagated it worldwide, and longest-prefix match sent the planet's YouTube traffic into Pakistan's blackhole at 18:47 UTC. YouTube counter-announced the same /24 (20:07), then split into two /25s to win on specificity (20:18); PCCW cut Pakistan Telecom off at 21:01. ~2¼ hours, fully documented by RIPE's route collectors. Teaches: a local censorship config became a global outage because BGP believes everyone; more-specifics are both the attack and the antidote.
- **Facebook erases itself — Oct 4, 2021.** (Full spine above.) A capacity-audit command with a buggy safety check downed the backbone; self-protective DNS servers withdrew their BGP routes; Facebook, Instagram, and WhatsApp vanished for ~6 hours; recovery needed physical data-center access and a megawatt-careful restart. Teaches: BGP is the existence layer; automation + correlated failure = self-amputation; keep your recovery tools off your own infrastructure.
- **Amazon Route 53 / MyEtherWallet hijack — April 24, 2018.** Attackers announced more-specifics of Route 53's address space (via eNet/AS10297), redirected DNS resolution for myetherwallet.com to a phishing server, and stole cryptocurrency (~$150k reported). A hijack of *DNS infrastructure* via BGP — two plumbing layers chained into a theft. Teaches: BGP attacks aren't just outages, they're interception; RPKI-invalid announcements like these are exactly what origin validation can drop. (Cited in Cloudflare's RPKI post.)
- **Optus, Australia — Nov 2023 (honorable mention).** A flood of route updates from a peer tripped safety limits in Optus routers, which shut down — a national telco offline ~12 hours, emergency calls affected. Teaches: even *defensive* BGP mechanisms (max-prefix limits) can cascade; worth a search for the Australian Senate inquiry if used.

## Visualization hooks
- **Route propagation as ink in water**: a world/AS-graph where one announcement diffuses node-to-node in seconds, each hop prepending its AS number — then run it backwards for the Facebook *withdrawal*, watching Facebook's color drain off the map.
- **The Facebook outage clock**: a 6-hour circular timeline — 15:40 withdrawals, 15:58 DNS gone, memes peak, engineers drive to data centers, 21:00 routes return, 21:20 DNS heals — annotated with the 30x resolver load curve.
- **Longest-prefix-match duel**: YouTube's /22 as a big flag, Pakistan's /24 as a smaller, sharper flag planted on top pulling all traffic arrows; YouTube's /25s as even sharper flags winning them back. Specificity as literal size.
- **The trust problem in one frame**: 78,000 nodes, each with a megaphone; one node announces "I am YouTube" and neighbors dutifully repeat it outward — no fact-checker in the loop.
- **AS-path accumulation**: a packet's route drawn as a train ticket getting stamped [13335] → [3356, 13335] → [7018, 3356, 13335] at each border.
- **RPKI as a deed registry overlay**: same AS graph, now some announcements carry a green signed seal (ROA-valid), forged ones flash red and get dropped at validating borders — but show the ~33% of routes still unsealed.
- **"Healthy but unreachable"**: Facebook's data centers glowing green inside a fence while every road to the fence is erased — the poster image for control-plane vs data-plane failure.
- **Customer/peer/transit economics**: the same destination reachable via three paths, router choosing the one that earns money over the one that costs money — policy beating shortest-path.

## Sources
- "More details about the October 4 outage" — Meta Engineering (Santosh Janardhan), 2021. The definitive first-party root cause: audit-tool bug, DNS withdrawal design, physical recovery. Primary. https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/
- "Understanding How Facebook Disappeared from the Internet" — Cloudflare blog, 2021. Outside-in BGP/DNS telemetry, exact prefixes and UTC timeline, 30x resolver load. Primary observer. https://blog.cloudflare.com/october-2021-facebook-outage/
- "YouTube Hijacking: A RIPE NCC RIS case study" — RIPE NCC, 2008. Minute-by-minute route-collector data on the 2008 hijack. Primary measurement. https://www.ripe.net/publications/news/youtube-hijacking-a-ripe-ncc-ris-case-study/
- "RPKI and BGP: our path to securing Internet Routing" — Cloudflare blog, 2018. ROAs, trust anchors, RTR, origin-vs-path limitation, 2018 Route 53 incident. Primary. https://blog.cloudflare.com/rpki/
- "BGP in 2025" — Geoff Huston, APNIC blog, Jan 2026. Table sizes, AS counts, growth trends; the annual reference. Primary measurement. https://blog.apnic.net/2026/01/08/bgp-in-2025/
- "Measuring BGP RPKI Route Origin Validation" — Cloudflare blog (rpki-updates-data), 2023+. ROV deployment measurement and Cloudflare Radar data. Primary. https://blog.cloudflare.com/rpki-updates-data/
- "RPKI Covers 67% of Routes..." — ipregistry.co, 2026. Current ROA coverage stat and RPKI limitations. Secondary; cross-check against NIST RPKI Monitor (https://rpki-monitor.antd.nist.gov/). https://ipregistry.co/blog/rpki-blind-spots/
- RFC 4271 — IETF, 2006. BGP-4 specification; skim §9 for route selection. Primary/RFC. https://datatracker.ietf.org/doc/html/rfc4271
