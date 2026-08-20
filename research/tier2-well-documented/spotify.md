# Spotify

## One-line hook
Spotify launched as a *peer-to-peer* network wearing a streaming service's clothes — then deleted the P2P layer, deleted its datacenters, and rebuilt itself as a data pipeline company on Google Cloud whose signature product (Discover Weekly) is a weekly batch job that regenerates a personal playlist for hundreds of millions of users.

## The core problem
Start music within ~250 milliseconds of a click — fast enough to feel local — for a catalog of 100M+ tracks, on 2008-era infrastructure a startup could afford; then, a decade later, turn half a trillion daily behavioral events into personalization that keeps ~700M monthly users listening. Spotify's story is two different hard problems in sequence: first *latency on a shoestring* (solved with P2P), then *personalization at scale* (solved with a cloud data platform).

## Architecture overview
**Then (2008-2014):** The desktop client was a hybrid: when you hit play, the first ~15 seconds came from Spotify's servers (or local cache) for instant start, while the client simultaneously searched an overlay P2P network — a tracker on Spotify's servers plus queries flooded to neighbor peers — and shifted the bulk of the download to other users' desktop clients. Caches were large, most listening was repeat listening, and by design only ~8.8% of audio bytes came from Spotify's own servers (2010 measurement), with median playback latency of 265 ms. Mobile clients never used P2P — which is one reason the P2P layer's days were numbered.

**Now (post-2016, all on Google Cloud):**
1. The client talks to Spotify's perimeter (access points/API gateways), behind which sit on the order of a thousand microservices owned by autonomous squads.
2. **Press play:** the client resolves track metadata and license, gets URLs for the encrypted audio files (Ogg Vorbis/AAC at multiple bitrates) which live in cloud storage and are served through CDN edges over HTTPS; the client caches aggressively and prefetches the next track for gapless playback. (Spotify even A/B-tested Google's BBR congestion control on this path in 2018.)
3. **Every user action emits an event** (play, skip, search, playlist-add). The event delivery system publishes each event type to its own Cloud Pub/Sub topic; ETL jobs (Dataflow and friends, ~2,500 GCE instances) deduplicate and land hourly partitions into Cloud Storage and BigQuery.
4. **ML pipelines** consume those datasets: collaborative filtering embeddings, NLP embeddings from crawled text, and audio CNN features are combined to generate candidates and rank tracks. Once a week, a batch pipeline regenerates Discover Weekly for every user; the playlist is just data written back to the playlist system.
5. Squad autonomy created thousands of services/components; Backstage — Spotify's internal service catalog and developer portal, open-sourced in 2020 — is the map that keeps the microservice sprawl navigable.

Component list (plain text):
- Clients (heavy caching, prefetch, gapless) → access points / API gateway
- ~1,000+ microservices on GCP (formerly 4 on-prem datacenters)
- Audio storage (cloud storage) + CDN delivery of encrypted Ogg/AAC
- Event delivery: per-event-type Cloud Pub/Sub topics → ETL → Cloud Storage/BigQuery
- Data/ML platform: Dataflow/Scio jobs, feature pipelines, weekly playlist-generation batch
- Historic: P2P overlay + tracker (desktop, 2008-2014); Kafka 0.7 + Hadoop event pipeline (pre-2016)
- Backstage developer portal (service catalog for squad-owned services)

## Signature ideas
**P2P as a startup's CDN (and its principled retirement).** Spotify's original trick: make users' desktops the content delivery network. Server-assisted start (first seconds from Spotify) hid P2P lookup latency; the overlay used a Spotify-run tracker plus peer gossip; caches made popular tracks swarm-available. It delivered ~90% of bytes without Spotify paying for them. By 2014, server/CDN costs had fallen, mobile (no P2P) dominated listening, and maintaining the P2P codebase wasn't worth it — Spotify shut the network down and went pure client-server. A perfect "right architecture for the era, retired without sentimentality" story.

**Event delivery as a product.** Spotify treats "get every play/skip/search event safely from any device to the data warehouse" as a first-class system with its own team. V1 (syslog + Kafka 0.7 + Hadoop) reliably moved 700K events/sec but had Hadoop as a single point of failure and no producer acknowledgments. V2 moved to Google Cloud Pub/Sub with a dedicated topic per event type — isolation so one busy event stream can't starve business-critical ones — and "liveness over lateness" hourly partitions. By 2019 it carried 8M events/sec at peak, 500B events and 350+ TB per day.

**Discover Weekly — personalization as a batch job.** Launched July 2015; 30 fresh songs every Monday for every user. Three model families feed it: collaborative filtering over the implicit-feedback play matrix (users who listen alike), NLP embeddings from crawling what the internet writes about songs/artists, and convolutional neural nets over the raw audio (letting brand-new tracks with no listening history be recommended). The insight that made it work: mine the 2+ billion user-made playlists as taste statements. 40M users and ~5B streams within the first year.

**The all-in cloud migration.** Announced Feb 2016 (~$450M committed over 3 years), Spotify moved ~1,200 microservices and 20K data jobs from four on-prem datacenters to GCP; services fully routed to GCP by May 2017; last datacenter closed in 2018. Strategic bet: buy infrastructure from Google, spend engineers on product — and the event-delivery rebuild (Kafka→Pub/Sub) was the pathfinder that proved managed services could beat self-run ones at Spotify's scale.

**Squads → microservice sprawl → Backstage.** Spotify's autonomous-squad model maps teams onto services (Conway's law, embraced deliberately): each squad builds, owns, and operates its slice. The architectural cost is discovery — thousands of components nobody can hold in their head — so Spotify built Backstage, a service catalog + "golden path" developer portal, and open-sourced it (Mar 2020); it's now a CNCF project used industry-wide. Org design as an architectural force, made visible.

## Key numbers
- 696M monthly active users, 276M Premium subscribers (Q2 2025 earnings, Jul 2025) — https://www.sec.gov/Archives/edgar/data/1639920/000114036125027654/ef20052398_ex99-1.htm (coverage: https://www.billboard.com/pro/spotify-q2-2025-earnings-subscribers-profit-revenue/)
- P2P era: only ~8.8% of audio bytes served from Spotify's servers; median playback latency 265 ms; 8M-track catalog (2010, IEEE P2P paper) — https://www.csc.kth.se/~gkreitz/spotify-p2p10/kreitz-spotify-p2p10.pdf
- ~80% of desktop traffic via P2P at peak (2011, reported at P2P shutdown, Apr 2014) — https://torrentfreak.com/spotify-starts-shutting-down-its-massive-p2p-network-140416/
- Event delivery v1: 700K+ events/sec through Kafka 0.7 + Hadoop (Feb 2016) — https://engineering.atspotify.com/2016/02/spotifys-event-delivery-the-road-to-the-cloud-part-i
- Event delivery on GCP: 8M events/sec peak, 500B events/day, 350+ TB/day, 500+ event types, ~2,500 GCE instances (Nov 2019, describing Q1 2019) — https://engineering.atspotify.com/2019/11/spotifys-event-delivery-life-in-the-cloud
- GCP migration: announced Feb 2016 with a reported ~$450M/3-year commitment; traffic fully on GCP May 2017; first on-prem DC closed Dec 2017, remaining three retired in 2018 (Dec 2019 retrospective) — https://engineering.atspotify.com/2019/12/views-from-the-cloud-a-history-of-spotifys-journey-to-the-cloud-part-1-2 (commitment figure: https://www.computerworld.com/article/1655983/how-spotify-migrated-everything-from-on-premise-to-google-cloud-platform.html)
- ~1,200 services and ~20,000 data jobs migrated by 100+ teams (2017-2018 migration, retold 2023) — https://startupgames.substack.com/p/the-epic-migration-of-spotify-to-google-cloud-platform-92372ae2d552
- Discover Weekly: launched Jul 2015; 40M listeners and ~5B track streams by May 2016 — https://musically.com/2016/05/25/spotify-discover-weekly-playlists-5bn-tracks/
- Discover Weekly: 2.3B+ hours streamed in its first five years (Jul 2020, Spotify Newsroom) — https://newsroom.spotify.com/2020-07-09/spotify-users-have-spent-over-2-3-billion-hours-streaming-discover-weekly-playlists-since-2015/
- 232M MAU at the time of the 2019 event-delivery post (Aug 2019) — https://engineering.atspotify.com/2019/11/spotifys-event-delivery-life-in-the-cloud

## Evolution timeline
- **2006-2008:** Founded in Stockholm; launches Oct 2008 with hybrid server + P2P desktop streaming, own datacenters, Ogg Vorbis audio.
- **2010:** Kreitz & Niemelä publish the P2P architecture paper — the canonical record of the design.
- **2011-2013:** Mobile explodes (no P2P there); backend grows into service-oriented architecture across 4 datacenters; Kafka+Hadoop data platform; squad/tribe org model formalized (2012 whitepaper).
- **Apr 2014:** P2P network shut down — pure client-server + CDN delivery from then on.
- **Jul 2015:** Discover Weekly launches; personalization becomes the flagship feature.
- **Feb 2016:** All-in GCP announcement; event delivery system rebuilt on Cloud Pub/Sub (the three-part blog series documents the decision).
- **May 2017-2018:** Traffic fully on GCP; all four on-prem datacenters closed.
- **2018:** IPO; data platform matures (Scio/Dataflow, BigQuery); BBR congestion-control experiments on audio delivery.
- **Mar 2020:** Backstage open-sourced (later CNCF); the squad-scale microservice catalog becomes an industry product.
- **2020s:** Podcasts then audiobooks broaden the catalog; recommendation stack evolves toward real-time and LLM-augmented features (AI DJ, 2023).

## Visualization hooks
- **The three sources of a song (2010):** animated playback bar showing bytes arriving from local cache, Spotify servers (the urgent first seconds), and dozens of peers (the bulk) — then a "2014" wipe where the peers vanish and a CDN takes their place.
- **Latency budget: 265 ms:** a stopwatch visualization decomposing click→sound — cache check, server-assisted start, P2P search overlapped in the background. Teaches "hide latency by starting from the fast source and switching."
- **The event river:** every tap in the app becomes a droplet; 8M droplets/sec flow into per-event-type Pub/Sub pipes (one pipe clogs — others keep flowing: isolation), then settle into hourly-partitioned lakes (Cloud Storage/BigQuery).
- **Kafka-era vs Pub/Sub-era side-by-side:** v1 with Hadoop as a single choke point (animate it failing and everything backing up) vs v2's independent per-type pipelines.
- **How Discover Weekly is cooked:** Sunday-night batch job assembly line — play matrix → CF embedding; web crawl → NLP embedding; waveform → CNN embedding; three vectors merge, rank, filter (already-heard tracks removed) → 30-song playlist stamped for Monday morning, times ~700M users.
- **A map of taste:** the classic embedding-space scatter — songs as points, a user as a region, Discover Weekly as the nearest unexplored points. Inherently visual.
- **Datacenters → cloud:** four glowing on-prem buildings (Stockholm/London/US) fading out 2016→2018 as workloads stream into GCP regions; overlay the counter "1,200 services, 20,000 data jobs."
- **Conway's law, drawn:** org chart of squads on the left, service graph on the right, with 1:1 edges — then zoom out to thousands of nodes and drop Backstage on top as the searchable map.

## Sources
- Kreitz, G. & Niemelä, F., "Spotify — Large Scale, Low Latency, P2P Music-on-Demand Streaming" — IEEE P2P Conference, 2010. The definitive record of the P2P design: tracker+gossip overlay, server-assisted start, 8.8%/265ms measurements. **Primary (peer-reviewed, by Spotify engineers).** https://www.csc.kth.se/~gkreitz/spotify-p2p10/kreitz-spotify-p2p10.pdf
- "Spotify Starts Shutting Down Its Massive P2P Network" — TorrentFreak, Apr 2014. The shutdown, with Spotify's own statement on why (enough server capacity). **Secondary with primary quotes.** https://torrentfreak.com/spotify-starts-shutting-down-its-massive-p2p-network-140416/
- "Spotify's Event Delivery — The Road to the Cloud, Parts I-III" — Spotify Engineering, Feb-Mar 2016. Old Kafka 0.7/Hadoop design, its failure modes, the Pub/Sub bet and load tests (Part II hammers Pub/Sub at 2M msg/sec). **Primary.** https://engineering.atspotify.com/2016/02/spotifys-event-delivery-the-road-to-the-cloud-part-i (Parts II/III linked from it)
- "Spotify's Event Delivery — Life in the Cloud" — Spotify Engineering, Nov 2019. The mature GCP system: 8M events/sec, 500B/day, per-type isolation, design principles. **Primary.** https://engineering.atspotify.com/2019/11/spotifys-event-delivery-life-in-the-cloud
- "Views From The Cloud: A History of Spotify's Journey to the Cloud, Part 1" — Spotify Engineering (Niklas Gustavsson, Chief Architect), Dec 2019. Migration strategy, timeline, datacenter closures. **Primary.** https://engineering.atspotify.com/2019/12/views-from-the-cloud-a-history-of-spotifys-journey-to-the-cloud-part-1-2
- "Spotify's journey to cloud: why Spotify migrated its event delivery system from Kafka to Google Cloud Pub/Sub" — Google Cloud Blog, 2016. Google-side account of the same decision. **Primary (vendor co-account).** https://cloud.google.com/blog/products/gcp/spotifys-journey-to-cloud-why-spotify-migrated-its-event-delivery-system-from-kafka-to-google-cloud-pubsub
- "How Spotify migrated everything from on-premise to Google Cloud Platform" — Computerworld, ~2018. The ~$450M/3yr commitment figure and migration mechanics. **Secondary.** https://www.computerworld.com/article/1655983/how-spotify-migrated-everything-from-on-premise-to-google-cloud-platform.html
- "Spotify Discover Weekly playlists have streamed nearly 5bn tracks" — Music Ally, May 2016. First-year adoption numbers (40M users, ~5B streams). **Secondary reporting Spotify figures.** https://musically.com/2016/05/25/spotify-discover-weekly-playlists-5bn-tracks/
- "Spotify Users Have Spent Over 2.3 Billion Hours Streaming Discover Weekly" — Spotify Newsroom, Jul 2020. Five-year milestone stats. **Primary.** https://newsroom.spotify.com/2020-07-09/spotify-users-have-spent-over-2-3-billion-hours-streaming-discover-weekly-playlists-since-2015/
- "How Does Spotify Know You So Well?" — Sophia Ciocca, Medium, 2017. The widely-cited explainer of the three Discover Weekly model families (CF, NLP, audio CNN), synthesizing Spotify talks (Chris Johnson's "From Idea to Execution" slides are the underlying primary). **Secondary; verify specifics against the talk slides.** https://medium.com/s/story/spotifys-discover-weekly-how-machine-learning-finds-your-new-music-19a41ab76efe
- "What the Heck is Backstage Anyway?" — Spotify Engineering, Mar 2020. Why microservice sprawl demanded a service catalog/developer portal. **Primary.** https://engineering.atspotify.com/2020/03/what-the-heck-is-backstage-anyway
- Spotify Q2 2025 earnings (Form 6-K) — SEC/Spotify, Jul 2025. 696M MAU, 276M subscribers. **Primary.** https://www.sec.gov/Archives/edgar/data/1639920/000114036125027654/ef20052398_ex99-1.htm
- "Smoother Streaming with BBR" — Spotify Engineering, Aug 2018. Evidence of the modern CDN/HTTPS audio path and congestion-control experimentation. **Primary.** https://engineering.atspotify.com/2018/08/smoother-streaming-with-bbr
