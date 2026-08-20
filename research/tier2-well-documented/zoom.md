# Zoom

## One-line hook
Zoom's servers never look at your video: they run a distributed SFU ("multimedia router") that only *forwards* encrypted streams — which is why, when the pandemic multiplied its load 30x in three months (10M → 300M daily meeting participants), it could survive by simply bursting more forwarding capacity into Oracle and AWS clouds.

## The core problem
Real-time, many-to-many video with humans in the loop: latency budgets of a few hundred milliseconds (no CDN tricks, no buffering — the opposite of Netflix/YouTube), dozens to hundreds of cameras per meeting, wildly heterogeneous devices and networks, and load that can spike faster than any capacity plan — most famously in March 2020, when the world's offices and schools moved onto Zoom in roughly two weeks. The architectural answer: never mix or transcode video on the server; route it.

## Architecture overview
Zoom splits into a **web/control plane** (user accounts, scheduling, auth, web portal — hosted in the public cloud, primarily AWS) and a **real-time media plane** (Zoom's own colocated datacenters plus burst capacity in public clouds).

**Joining a meeting, end to end:**
1. The client authenticates via the web infrastructure and asks to join a meeting. It is directed (over HTTPS) to the nearest of Zoom's datacenters, interconnected by private links.
2. Inside each datacenter sit **Meeting Zones** — server clusters with a **Zone Controller** (admission, load monitoring, assignment) and **Multimedia Routers (MMRs)**, which host the actual meetings. The Zone Controller picks an MMR for the new meeting or maps the client onto the existing one.
3. Each client sends **one uplink** — its own encoded video in multiple simulcast resolutions plus audio — preferably over UDP (falling back to TCP/TLS/443 through restrictive firewalls). The MMR **selectively forwards** streams: the active speaker's higher-resolution stream to everyone, low-resolution thumbnails for the gallery, each sized to the receiver's window layout and measured bandwidth. No decoding, mixing, or re-encoding happens server-side.
4. All encode/decode work happens on clients; the client adapts resolution/frame rate to CPU and network, with jitter buffers and error correction smoothing the last mile.
5. For geo-distributed meetings, each participant connects to their *nearest* Meeting Zone and MMRs cascade the streams between zones over Zoom's private backbone, keeping every participant's first hop short.
6. Cloud recordings, dial-in (PSTN) gateways, and the web (browser) client are bridge services that attach to the same MMR meeting.

Component list (plain text):
- Clients (native apps do the heavy lifting: encode/decode, simulcast, adaptation)
- Web/control plane in public cloud: auth, scheduling, portal, telemetry dashboard
- ~17-21 colocated datacenters (2020) + private interconnects (Equinix fabric)
- Meeting Zone = Zone Controller + Multimedia Router (MMR) cluster
- Distributed SFU forwarding: 1 uplink in, N tailored downlinks out; cross-zone MMR cascading
- Burst capacity: AWS (primary), Oracle Cloud (2020-2022 era) for MMR fleets
- Gateways: PSTN dial-in, H.323/SIP room connectors, browser/WebRTC bridge, cloud recording

## Signature ideas
**Multimedia routing: SFU over MCU.** Legacy conferencing used MCUs (Multipoint Control Units) that decoded every participant's video, composited one picture, and re-encoded it per receiver — compute-heavy, quality-lossy, and capped around 100 participants per box. Zoom's MMR is a distributed Selective Forwarding Unit: it takes one uplink per client and forwards multiple downlinks, moving the encode/decode burden to the endpoints. Zoom claimed ~15x the participant capacity of a typical MCU, which is how "500-person meeting" became a checkbox rather than a hardware purchase (Zoom blog, 2019).

**Servers that scale with bandwidth, not pixels.** Because MMRs only shuffle encrypted packets, server cost grows with network throughput rather than video-processing FLOPs. That made the 2020 emergency scale-up tractable: the bottleneck was "more boxes forwarding bytes," a commodity resource that public clouds could supply overnight — no exotic media hardware needed.

**The colo + multi-cloud hybrid.** Zoom's steady-state real-time traffic runs from its own colocated datacenters (architected to run at ~50% of peak so any site can absorb another's load), with the web tier and burst media capacity in public cloud. In 2020 it famously added Oracle Cloud (millions of participants; ~7 PB/day through OCI within weeks of the deal) alongside AWS — which "turned up" thousands of servers per day for Zoom — then signed AWS as its preferred cloud provider that November. A clean case study in bursting: own the baseline, rent the spike.

**Geography-aware meeting placement — and its failure mode.** Meetings are hosted in the zone nearest participants, with cross-zone cascades for global calls. In April 2020, researchers at Citizen Lab found some non-China meetings being routed through China-based servers (and weaknesses in the then-AES-128-ECB encryption); Zoom responded within days by letting paying customers choose/exclude datacenter regions and, in Zoom 5.0 (May 2020), upgrading to AES-256-GCM — and later end-to-end encryption (Oct 2020). Routing topology became a trust and geopolitics issue, not just a latency one.

**Client-side intelligence.** Zoom's native clients use a proprietary protocol stack (not vanilla WebRTC) tuned for bad networks: simulcast layers, aggressive UDP-first transport with TCP/443 fallback that traverses almost any firewall, forward error correction, and graceful degradation (video sacrificed before audio). Much of Zoom's "it just works" reputation is endpoint engineering, with the server kept deliberately dumb.

## Key numbers
- 10M peak daily meeting participants (Dec 2019) → 200M (Mar 2020) → 300M+ (Apr 2020): ~30x in ~3 months (Zoom blog, Apr 2020; clarified as "meeting participants," not unique users, Apr 30, 2020) — https://www.zoom.com/en/blog/a-message-to-our-users/ and https://www.cnbc.com/2020/04/30/zoom-walks-back-claims-it-has-300-million-daily-active-users.html
- ~7 petabytes/day flowing through Oracle Cloud Infrastructure for Zoom within weeks of the April 2020 deal ("millions of meeting participants" moved to OCI) (Apr 2020) — https://www.computerweekly.com/news/252482328/Zoom-chooses-Oracle-IaaS-as-core-cloud-infrastructure-provider
- 17 datacenter locations worldwide, interconnected privately and architected to run at ~50% of peak capacity for failover headroom (2020) — https://www.datacenterfrontier.com/cloud/article/11428986/inside-zoom8217s-infrastructure-scaling-up-massively-with-colo-and-cloud
- Multimedia routing claimed to support ~15x more participants than a standard MCU (which typically capped below ~100); Zoom raised Business-tier meeting capacity to 300 and offered up to 500-1,000 via large-meeting add-ons (Jun 2019) — https://www.zoom.com/en/blog/zoom-can-provide-increase-industry-leading-video-capacity/
- Annualized meeting-minutes run rate exceeded 3.3 trillion (Zoom Q3 FY2021 earnings, Nov-Dec 2020) — https://www.businessofapps.com/data/zoom-statistics/
- Multi-year AWS "preferred cloud provider" agreement announced Nov 30, 2020, after AWS spun up thousands of servers a day for Zoom during the 2020 surge — https://www.computerweekly.com/news/252492929/Zoom-signs-multi-year-preferred-cloud-provider-deal-with-AWS
- Global infrastructure whitepaper (Sept 2019) documents the datacenter + Meeting Zone + MMR design pre-pandemic — https://zoomgov.com/docs/doc/Zoom_Global_Infrastructure.pdf

## Evolution timeline
- **2011-2013:** Founded by Eric Yuan (ex-Cisco/WebEx) explicitly to rebuild conferencing around cloud-native multimedia routing rather than MCU hardware; launches 2013.
- **2013-2019:** Steady growth on the colo-datacenter + Meeting Zone architecture; proprietary UDP-first client protocol; H.323/SIP and PSTN gateways bolt legacy systems onto the MMR core; IPO April 2019; ~17 datacenters plus AWS/Azure for the web tier by late 2019 (global infrastructure whitepaper, Sept 2019).
- **Dec 2019:** ~10M peak daily meeting participants.
- **Mar-Apr 2020:** COVID lockdowns: 200M (March) then 300M+ (April) daily meeting participants; capacity emergency-bursted into AWS (thousands of servers/day) and, from late April, Oracle Cloud (~7 PB/day). Free tier for K-12 schools.
- **Apr-May 2020:** Trust crisis and response: Citizen Lab report (China routing, weak AES-ECB crypto) → 90-day security freeze; datacenter region opt-out for paid accounts (Apr 18); Zoom 5.0 ships AES-256-GCM (May); Keybase acquisition; end-to-end encryption GA Oct 2020.
- **Nov 2020:** AWS becomes preferred cloud provider (Oracle relationship later wound down); annualized meeting minutes pass 3.3 trillion.
- **2021-2025:** Post-pandemic normalization; platform expands (Zoom Phone, Contact Center, Rooms); AI Companion features (2023+) layer ML services onto the same meeting fabric; architecture remains colo-core + cloud-burst with regionalized routing controls (documented in Zoom's current "Architected for Reliability" technical library).

## Visualization hooks
- **MCU vs SFU, side by side:** left, an MCU decoding 9 tiles, compositing, re-encoding (CPU meters maxed, one stream out); right, an MMR passing 9 encrypted streams through untouched (CPU idle, streams fanned out per viewer). The single best drawable idea in the system.
- **One uplink, tailored downlinks:** a participant's camera emitting 3 simulcast layers (high/medium/thumbnail); the MMR picking the large stream for the active-speaker view and thumbnails for the gallery — redrawn live as someone starts talking.
- **The 30x cliff:** a line chart Dec 2019 → Apr 2020 (10M → 200M → 300M daily participants) annotated with what turned on when: AWS burst, Oracle deal (7 PB/day), school free tier. Almost vertical.
- **Colo core, cloud balloon:** Zoom's ~17 datacenters as a solid ring running at 50% capacity; in March 2020 cloud regions inflate around them like airbags absorbing the overflow — deflating partially in 2022.
- **A meeting spanning the planet:** three participants (Tokyo, Berlin, São Paulo) each connecting to their nearest Meeting Zone, MMRs cascading between zones over private links — versus the naive design where everyone hairpins through one US server.
- **Firewall escape sequence:** animated fallback ladder — UDP blocked → TCP → TLS on port 443 ("looks like ordinary HTTPS, so it always gets through").
- **The routing-as-trust map:** April 2020 world map showing a meeting's media unexpectedly transiting a China datacenter, then the "choose your regions" control fencing routes — infrastructure topology becoming a privacy feature.
- **Latency budget vs the streamers:** a horizontal bar comparing Netflix (seconds of buffer allowed) vs Zoom (~200 ms mouth-to-ear) — why nothing in this system can be cached and everything must be routed.

## Sources
- "Here's How Zoom Provides Industry-Leading Video Capacity" — Zoom blog, Jun 2019. The official explanation of multimedia routing vs MCU, the 15x capacity claim, client-side encode/decode. **Primary.** https://www.zoom.com/en/blog/zoom-can-provide-increase-industry-leading-video-capacity/
- "Zoom Global Infrastructure" whitepaper — Zoom (zoomgov), Sept 2019. Pre-pandemic architecture: datacenters, Meeting Zones, Zone Controller + MMR, cloud/web tier split. **Primary.** https://zoomgov.com/docs/doc/Zoom_Global_Infrastructure.pdf
- "Zoom: Architected for Reliability" — Zoom Technical Library, current. Present-day official description of the distributed architecture and failover design. **Primary.** https://library.zoom.com/admin-corner/architecture-and-design/zoom-architected-for-reliability
- "A Message to Our Users" — Eric Yuan, Zoom blog, Apr 1, 2020. The 10M→200M numbers and the 90-day security plan, from the CEO. **Primary.** https://www.zoom.com/en/blog/a-message-to-our-users/
- "Zoom walks back claims it has 300 million daily active users" — CNBC, Apr 30, 2020. The participants-vs-users correction; keeps the 300M figure honest. **Secondary.** https://www.cnbc.com/2020/04/30/zoom-walks-back-claims-it-has-300-million-daily-active-users.html
- "Inside Zoom's Infrastructure: Scaling Up Massively With Colo and Cloud" — Data Center Frontier, 2020. The 17 datacenters, Equinix interconnection, 50%-of-peak headroom rule, colo+cloud burst strategy. **Secondary (trade press with Zoom interviews).** https://www.datacenterfrontier.com/cloud/article/11428986/inside-zoom8217s-infrastructure-scaling-up-massively-with-colo-and-cloud
- "Zoom chooses Oracle IaaS as core cloud infrastructure provider" — Computer Weekly, Apr 2020. The Oracle deal: 7 PB/day, millions of participants on OCI. **Secondary reporting primary press statements.** https://www.computerweekly.com/news/252482328/Zoom-chooses-Oracle-IaaS-as-core-cloud-infrastructure-provider
- "Zoom signs multi-year preferred cloud provider deal with AWS" — Computer Weekly, Dec 2020. The AWS relationship and its role in the 2020 surge. **Secondary.** https://www.computerweekly.com/news/252492929/Zoom-signs-multi-year-preferred-cloud-provider-deal-with-AWS
- "A Study of Zoom's Video Conferencing Architecture & System Design" — CometChat engineering blog, ~2021. Clear technical walkthrough of Meeting Zones, Zone Controllers, MMR-as-distributed-SFU, protocol fallbacks. **Secondary; cross-checked against Zoom's whitepaper.** https://www.cometchat.com/blog/zoom-video-technology-architecture
- "Move Fast & Roll Your Own Crypto: A Quick Look at the Confidentiality of Zoom Meetings" — Citizen Lab (U. of Toronto), Apr 3, 2020. The China-routing and AES-128-ECB findings that reshaped Zoom's routing controls and crypto. **Primary (independent research).** https://citizenlab.ca/2020/04/move-fast-roll-your-own-crypto-a-quick-look-at-the-confidentiality-of-zoom-meetings/
- "Zoom Revenue and Usage Statistics" — Business of Apps, updated 2024+. Aggregates Zoom's disclosed metrics incl. the 3.3T annualized meeting minutes (Q3 FY2021). **Secondary (aggregator of primary disclosures).** https://www.businessofapps.com/data/zoom-statistics/
