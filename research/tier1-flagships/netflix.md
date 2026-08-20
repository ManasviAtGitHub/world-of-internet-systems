# Netflix

## One-line hook
Netflix is really two systems glued together at the Play button: a control plane of microservices on AWS that decides *what* you watch, and a private CDN of servers living inside ISPs (Open Connect) that delivers *every single video byte* — and the company invented chaos engineering to keep the first half alive.

## The core problem
Stream high-quality video to 300+ million paying members on thousands of device types, at a scale where Netflix alone is roughly 15% of the world's downstream internet traffic — while keeping start times under a couple of seconds, avoiding rebuffering on flaky home networks, and surviving constant hardware/zone/region failures in the cloud. The bandwidth bill and the reliability problem are both existential: solving them produced Open Connect, per-title encoding, and chaos engineering.

## Architecture overview
Netflix splits cleanly into a **control plane** and a **data plane**, and they hand off exactly once — when you press Play.

**Control plane (AWS, 3+ regions: N. Virginia, Oregon, Dublin):** hundreds of microservices handle sign-up, authentication, profiles, search, recommendations, A/B tests, billing, and playback decisioning. Traffic enters through the Zuul edge gateway, services find each other via Eureka service discovery, data lives in Cassandra clusters fronted by the EVCache memcached tier, and events flow through Kafka into the analytics stack. The device-facing API layer is today a federated GraphQL gateway routing to Domain Graph Services owned by individual teams.

**Data plane (Open Connect):** Netflix's own CDN — purpose-built FreeBSD + NGINX servers called Open Connect Appliances (OCAs), deployed in two places: (1) at internet exchange points (IXPs) peering directly with ISPs, and (2) *embedded inside ISP networks*, given to qualifying ISPs for free. OCAs only serve static video/image bytes; they hold no customer logic.

**End-to-end: pressing Play**
1. Long before you press Play, the title was ingested: the source master is validated, split into chunks, and encoded in parallel on hundreds of thousands of CPUs into dozens of codec/bitrate/resolution combinations per title (plus audio tracks and subtitles).
2. Every night during off-peak "fill windows," each OCA asks AWS which titles it should carry — driven by popularity prediction for its region — and pre-positions tomorrow's likely viewing so the fill traffic never competes with peak streaming.
3. You press Play. The request goes to AWS: the Playback Apps service checks entitlements and picks the right files for your device/bandwidth; the steering service uses OCA health reports, file availability, and your network location (IP/ISP data) to compute the best delivery sites.
4. AWS returns URLs for up to ten ranked OCAs. Your device probes them, picks the best, and starts pulling video over HTTPS with adaptive bitrate switching — the client keeps measuring and can shift bitrate or fail over to another OCA mid-stream.
5. Telemetry (start time, rebuffers, quality shifts) streams back into AWS through the Kafka-based event pipeline to feed QoE monitoring and the next round of popularity prediction.

Component list (plain text):
- Client apps (TV/mobile/web) with adaptive-bitrate player
- Zuul edge gateway → federated GraphQL API layer → microservices (AWS)
- Eureka (service discovery), Hystrix-era circuit breaking → resilience4j/adaptive concurrency today
- EVCache (distributed memcached tier), Cassandra (source-of-truth storage), Kafka (events)
- Encoding pipeline (chunked, parallel, per-title/per-shot optimized; VMAF quality metric)
- Open Connect: IXP-site OCAs + ISP-embedded OCAs, nightly proactive fill, steering service in AWS
- Chaos tooling: Chaos Monkey → Simian Army → FIT/ChAP failure injection

## Signature ideas
**Open Connect — a CDN made of free boxes inside ISPs.** Instead of paying commercial CDNs, Netflix builds its own storage appliances and ships them free to ISPs, who host them because they cut the ISP's own transit costs. Video is served from inside the user's own ISP network; by 2016 ~90% of traffic was delivered over direct Netflix↔ISP connections and 100% of video went through Open Connect. Per-server throughput went from 8 Gbps (2012) to 90+ Gbps (2016) and later to 400 Gbps in lab work, largely via FreeBSD kernel and NUMA engineering.

**Chaos engineering / Simian Army.** Born from the 2008 lesson that hardware fails: Chaos Monkey randomly terminates production instances during business hours so engineers are forced to build services that survive instance death. The 2011 Simian Army generalized it — Latency Monkey (inject delays), Conformity/Janitor/Security Monkeys (hygiene), Chaos Gorilla (kill an availability zone), Chaos Kong (evacuate a whole AWS region). Netflix practices region evacuation monthly and can drain a region in minutes. This grew into the discipline now called chaos engineering.

**Per-title and per-shot encoding.** Instead of one fixed bitrate ladder for all content, Netflix analyzes each title (Dec 2015) — simple animation needs a fraction of the bits an action film needs at the same perceptual quality — and later moved to per-shot optimization with the Dynamic Optimizer (2018), encoding each shot at its own optimal settings and stitching the results, guided by their open-sourced perceptual metric VMAF. Reported savings are on the order of 20-30% bitrate at equal quality — which at Netflix scale is a measurable share of the whole internet.

**Netflix OSS: the microservices starter kit for everyone else.** During its 7-year cloud migration Netflix open-sourced its infrastructure: Zuul (edge gateway), Eureka (service discovery), Hystrix (circuit breakers/bulkheads), Ribbon, Spinnaker. Spring Cloud Netflix made these the de-facto microservices toolkit of the 2010s JVM world. Today Hystrix is retired (maintenance mode since late 2018, resilience4j and adaptive concurrency limits recommended), Zuul 2 is async on Netty, and the API layer is federated GraphQL (2020) — a tidy story of an architecture generation rising and being replaced.

**EVCache + Cassandra: the two-layer data tier.** Cassandra (chosen for multi-region, always-writable availability) is the durable store — Netflix demonstrated 1M+ writes/sec on 288 AWS nodes back in 2011. EVCache, a sharded, cross-region-replicated memcached layer, absorbs the read storm in front of it; a page load touches caches hundreds of times, and the tier grew from tens of millions of ops/sec (2016) to hundreds of millions.

**Predictive caching: ship the bytes before anyone asks.** Netflix knows tonight what will be watched tomorrow, so OCAs pull new/popular content during ISP off-peak windows. The CDN is mostly *pre-filled*, not demand-filled — inverting the classic CDN cache-miss model and making peak-hour delivery almost purely local reads.

## Key numbers
- 301.6M paid memberships; record +19M in Q4 2024 (announced Jan 2025) — https://variety.com/2025/tv/news/netflix-subscribers-300-million-q4-2024-1236280419/
- ~15% of global downstream internet traffic, #1 single application (Sandvine Global Internet Phenomena, 2023) — https://www.applogicnetworks.com/inthenews/netflix-eats-up-15-of-global-downstream-traffic
- Cloud migration took 7 years, completed Jan 2016; 8x more streaming members in 2016 vs 2008; viewing grew ~3 orders of magnitude 2008-2016 (Feb 2016) — https://about.netflix.com/en/news/completing-the-netflix-cloud-migration
- Open Connect (as of Mar 2016): ~1,000 locations; ~90% of traffic via direct ISP connections; 100% of video via Open Connect; per-server throughput 8 Gbps (2012) → 90+ Gbps (2016); 125M+ hours viewed/day — https://about.netflix.com/en/news/how-netflix-works-with-isps-around-the-globe-to-deliver-a-great-viewing-experience
- 1,000+ ISP partners with embedded OCAs; 60+ data-center/IXP footprints (Open Connect site, accessed 2026) — https://openconnect.netflix.com/en/
- 2017 snapshot: 110M+ members, 200+ countries, ~1B hours watched/week, several hundred thousand EC2 instances, region evacuation in ~6 minutes (Dec 2017) — https://highscalability.com/netflix-what-happens-when-you-press-play/
- Encoding scale (2017): ~300,000 CPUs for the media pipeline; Stranger Things S2 produced 9,570 output files and took 190,000 CPU-hours (Dec 2017) — https://highscalability.com/netflix-what-happens-when-you-press-play/
- Cassandra benchmark: 1.1M+ writes/sec on 288 nodes in AWS (Nov 2011) — https://netflixtechblog.com/benchmarking-cassandra-scalability-on-aws-over-a-million-writes-per-second-39f45f066c9e
- EVCache (2016): tens of millions of ops/sec, thousands of memcached instances, 100+ TB cached (Jan 2016) — https://netflixtechblog.com/caching-for-a-global-netflix-7bcc457012f1
- Per-shot Dynamic Optimizer: ~30% bitrate savings vs prior per-title encodes, depending on codec/metric (Mar 2018) — https://netflixtechblog.com/dynamic-optimizer-a-perceptual-video-encoding-optimization-framework-e19f1e3a277f
- Live streaming record: 65M peak concurrent streams (38M in the US) for Paul vs. Tyson, ~108M average-minute global viewers (Nov 2024) — https://about.netflix.com/en/news/jake-paul-vs-mike-tyson-over-108-million-live-global-viewers

## Evolution timeline
- **1998-2007:** DVD-by-mail; monolithic Java app + Oracle in Netflix's own datacenter.
- **2007:** Streaming launches (initially via commercial CDNs: Akamai/Limelight/Level 3).
- **Aug 2008:** Major database corruption halts DVD shipping for 3 days → decision to move to cloud and rebuild as distributed, stateless, cloud-native services.
- **2009-2012:** Piece-by-piece migration to AWS; Cassandra replaces Oracle/SimpleDB for member-facing data; Chaos Monkey (2010) and Simian Army (2011) institutionalize failure testing; EVCache introduced (2011).
- **2012:** Open Connect announced — Netflix starts running its own CDN and phasing out commercial CDNs.
- **2012-2015:** Netflix OSS era: Eureka, Hystrix, Zuul, Ribbon open-sourced; Spring Cloud Netflix spreads the patterns industry-wide.
- **Dec 2015:** Per-title encode optimization replaces the fixed bitrate ladder.
- **Jan 2016:** Cloud migration declared complete; simultaneous launch in 130 new countries (190 total); VMAF perceptual metric open-sourced (Jun 2016).
- **2018:** Zuul 2 (async, Netty); Dynamic Optimizer brings shot-based encoding; Hystrix enters maintenance mode (resilience4j recommended).
- **2020-2021:** API layer rebuilt as federated GraphQL (Domain Graph Services); Titus container platform and Spinnaker mature as the deploy substrate.
- **2022-2025:** Ads tier (2022); live events push the architecture into low-latency territory — Chris Rock live (2023), Paul vs. Tyson 65M concurrent streams (2024), NFL Christmas games (2024) — plus games and cloud gaming streaming.

## Visualization hooks
- **The two-plane handoff:** a split diagram — AWS control plane on the left, Open Connect data plane on the right, with exactly one arrow between them labeled "Play." Animate a request crossing it once and video bytes never touching AWS.
- **World map of a hidden CDN:** dots for ~1,000+ OCA sites inside ISP networks vs. just 3 AWS regions — the visual contrast between "brain" (3 sites) and "muscle" (thousands).
- **The nightly fill window:** a 24-hour clock animation — daytime: OCAs serve; 2-6 AM: OCAs quietly pull tomorrow's predicted hits from peers/origin. Before/after of demand-filled CDN (cache misses at peak) vs. pre-filled CDN.
- **Press Play, frame by frame:** animated sequence of the 5-step flow — auth in AWS → steering returns 10 OCA URLs → client probes → best OCA streams → ABR ladder switching as bandwidth wobbles.
- **The bitrate ladder, before/after per-title:** two step-charts — fixed ladder identical for cartoon and action film vs. per-title convex-hull ladders hugging each title's rate-quality curve; shade the saved bandwidth area.
- **Chaos Monkey as a game:** a grid of healthy instances; a monkey randomly kills one; traffic instantly reroutes (Eureka + load balancer); scale up to Chaos Gorilla (a whole zone goes dark) and Chaos Kong (a region drains in ~6 minutes on a US map).
- **One title → 1,200 files:** fan-out diagram of a single master encoding into codecs × resolutions × bitrates × audio languages × subtitles (Stranger Things S2: 9,570 files).
- **Netflix vs. the internet:** area chart of downstream traffic share (~15% globally) — one company as a continent of the internet's traffic map.

## Sources
- "Completing the Netflix Cloud Migration" — about.netflix.com, Feb 2016. The official end-of-migration post: 7-year timeline, 2008 corruption incident, 8x member growth, what stayed out of AWS. **Primary.** https://about.netflix.com/en/news/completing-the-netflix-cloud-migration
- "How Netflix Works With ISPs Around the Globe to Deliver a Great Viewing Experience" — about.netflix.com, Mar 2016. Open Connect scale: ~1,000 locations, 90% direct delivery, 8→90 Gbps per server, 125M hours/day. **Primary.** https://about.netflix.com/en/news/how-netflix-works-with-isps-around-the-globe-to-deliver-a-great-viewing-experience
- Open Connect overview + appliances — openconnect.netflix.com, current. Deployment models (embedded vs. SFI/IXP), free appliances, fill process, partner requirements. **Primary.** https://openconnect.netflix.com/en/ and https://openconnect.netflix.com/Open-Connect-Overview.pdf
- "The Netflix Simian Army" — Netflix TechBlog, Jul 2011. Names and roles of all the monkeys; the design-for-failure rationale. **Primary.** https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116
- "Per-Title Encode Optimization" — Netflix TechBlog, Dec 2015. Why fixed ladders waste bits; per-title analysis and convex hull ladder construction. **Primary.** https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2
- "Dynamic optimizer — a perceptual video encoding optimization framework" — Netflix TechBlog, Mar 2018. Shot-based encoding, VMAF-guided optimization, ~30% savings. **Primary.** https://netflixtechblog.com/dynamic-optimizer-a-perceptual-video-encoding-optimization-framework-e19f1e3a277f
- "Benchmarking Cassandra Scalability on AWS — Over a million writes per second" — Netflix TechBlog, Nov 2011. The 288-node, 1.1M writes/sec benchmark that made Cassandra-on-EC2 credible. **Primary.** https://netflixtechblog.com/benchmarking-cassandra-scalability-on-aws-over-a-million-writes-per-second-39f45f066c9e
- "Caching for a Global Netflix" / "Announcing EVCache" — Netflix TechBlog, Jan 2016 / 2013. EVCache design: sharded memcached, cross-region replication, ops/sec scale. **Primary.** https://netflixtechblog.com/caching-for-a-global-netflix-7bcc457012f1
- "Netflix: What Happens When You Press Play?" — HighScalability (Todd Hoff), Dec 2017. The best end-to-end narrative: encoding pipeline, OCA steering (10 URLs), nightly fill, 2017 scale stats. **Secondary but well-sourced; verified against Netflix docs.** https://highscalability.com/netflix-what-happens-when-you-press-play/
- "How Netflix Scales its API with GraphQL Federation (Part 1)" — Netflix TechBlog, Nov 2020. The post-REST API architecture: gateway + Domain Graph Services. **Primary.** https://netflixtechblog.com/how-netflix-scales-its-api-with-graphql-federation-part-1-ae3557c187e2
- Netflix/Hystrix GitHub README — Netflix OSS, status updated Nov 2018. Confirms maintenance mode and the recommendation of resilience4j / adaptive concurrency limits. **Primary.** https://github.com/Netflix/Hystrix
- "Netflix Adds Nearly 19 Million Subscribers…" — Variety, Jan 2025. 301.6M members, record Q4 2024 adds; last quarterly subscriber disclosure. **Secondary (reporting Netflix earnings, reliable).** https://variety.com/2025/tv/news/netflix-subscribers-300-million-q4-2024-1236280419/
- "Jake Paul vs. Mike Tyson… Over 108 Million Live Global Viewers" — about.netflix.com, Nov 2024. Live-streaming records: 65M peak concurrent streams. **Primary.** https://about.netflix.com/en/news/jake-paul-vs-mike-tyson-over-108-million-live-global-viewers
- Sandvine Global Internet Phenomena coverage — applogicnetworks.com (Sandvine), 2023. Netflix ~15% of global downstream traffic. **Primary (vendor measurement).** https://www.applogicnetworks.com/inthenews/netflix-eats-up-15-of-global-downstream-traffic
- FreeBSD Foundation Netflix case study — freebsdfoundation.org, 2020. Why OCAs run FreeBSD; kernel work behind extreme per-server throughput. **Primary (partner org).** https://freebsdfoundation.org/wp-content/uploads/2020/10/netflixcasestudy_final.pdf
