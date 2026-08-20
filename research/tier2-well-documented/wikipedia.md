# Wikipedia (Wikimedia infrastructure)

## One-line hook
A top-10 website serving hundreds of billions of pageviews a year runs on roughly a thousand physical servers and a hosting bill of a few million dollars — because when reads outnumber writes ~1000:1, caching *is* the architecture. And uniquely, every layer of it is publicly documented.

## The core problem
Serve encyclopedia pages to the whole planet, on a nonprofit budget, with a stack that volunteers can inspect. The workload is extreme but merciful: ~296 billion pageviews in 2024 (~10,000/sec average) against ~16 edits/sec across all projects — a read:write ratio in the high hundreds to one. The hard parts: (1) a page is *not* static — wikitext must be parsed into HTML, and one edit to a popular template can invalidate millions of pages; (2) any page can be edited at any moment and readers expect to see the change quickly, so caches must be *purged*, not just expired; (3) do all of this with a fraction of the money and headcount of commercial peers. Unlike Google or Amazon, none of this is secret: wikitech.wikimedia.org documents the production stack down to individual server roles.

## Architecture overview
The design is a funnel: billions of requests enter, and each successive cache layer strips away traffic so only a trickle reaches the databases.

**Read path (the common case):**
1. **GeoDNS (gdnsd)** sends the user to the nearest of 7 sites: two full application datacenters — eqiad (Ashburn, VA) and codfw (Carrollton, TX) — plus caching-only edge PoPs in Amsterdam (esams), San Francisco (ulsfo), Singapore (eqsin), Marseille (drmrs), and São Paulo (magru) (Meta "Wikimedia servers", accessed 2026).
2. **LVS/PyBal load balancers** spread traffic across the edge cluster; **HAProxy** terminates TLS/HTTP2 and rate-limits.
3. **Varnish (frontend cache, in-memory)** — full HTTP responses for hot objects; absorbs request storms for popular articles. Frontend TTLs are capped at 1 day.
4. **Apache Traffic Server (backend cache, on-disk)** — misses are hashed by URL so each ATS node owns a slice of URL space; up to 24h retention. Two logical clusters: `cache_text` (article HTML, APIs) and `cache_upload` (images/thumbnails from Swift, maps) (Wikitech CDN page, accessed 2026). Roughly 90% of incoming requests are answered at this CDN layer without touching application servers (~1,300 bare-metal servers total; 36C3 talk, 2019).
5. **MediaWiki application layer** — the same PHP monolith anyone can download, running on Kubernetes at WMF. Before rendering from scratch it consults the **ParserCache** (already-rendered HTML of wikitext, retained ~weeks, stored in dedicated DBs) and **Memcached/mcrouter** (objects: user sessions, localization, computed fragments).
6. **MariaDB** — only now. Wikis are grouped into sections (s1 = English Wikipedia, s2…s8, plus x1/x2 and ParserCache clusters), each a primary with replicas; article revision text lives compressed in a separate "external storage" cluster. Search queries go to OpenSearch (CirrusSearch); media files live in Swift.

**Write path (the rare case):** An edit takes the same route but cannot be served from cache: MediaWiki writes the new revision to the section primary (always in the primary DC), invalidates ParserCache/memcached entries, and — the signature move — sends **explicit purge messages** to every CDN node so the old HTML vanishes worldwide within seconds. Template edits fan out via the job queue, re-parsing affected pages lazily. Since ~2022 MediaWiki runs **active-active**: both eqiad and codfw serve reads, writes are routed to the primary DC, and the primary role is swapped in regular switchover drills (Wikitech Multi-DC, 2022).

**Component list (plain text):**
- gdnsd GeoDNS → LVS/PyBal L4 load balancing → HAProxy TLS termination
- Varnish in-memory frontend cache (hot objects, chash by popularity)
- Apache Traffic Server on-disk backend cache (URL-hashed shards)
- MediaWiki (PHP) on Kubernetes; ParserCache; Memcached/mcrouter
- MariaDB replicated sections s1–s8 + external storage (revision text)
- Swift object storage + Thumbor (media/thumbnails); OpenSearch/CirrusSearch (search)
- Kafka + job queue (async re-parses, purges); CDN purge fan-out
- 2 application DCs (eqiad, codfw, active-active reads) + 5 caching PoPs

## Signature ideas
- **The read:write funnel.** Public numbers make the textbook lesson concrete: ~296B pageviews vs ~500M edits a year (2024 figures) is roughly 600 reads per write, and after the CDN absorbs ~90%, the "real" dynamic workload is a small fraction of a percent of nominal traffic. The database tier is sized for the funnel's narrow end, not its mouth — which is why a few hundred DB servers suffice for a top-10 site.
- **Purge, don't just expire.** Most CDNs accept staleness; Wikipedia can't ("vandalism must disappear *now*"). Every cached URL is actively invalidated on edit via purge messages fanned out to all cache nodes, layered on top of conservative TTL caps (1 day frontend / 24h backend) as a safety net. This "event-driven invalidation over TTL backstop" pattern is one of the most instructive cache designs in public documentation (Wikitech CDN, accessed 2026).
- **Two-tier edge caches with different physics.** Varnish keeps *popular* objects in RAM, replicated on every frontend node, so a viral article is served from every machine at memory speed. ATS keeps the *long tail* on disk, hash-partitioned so each URL exists on one node — capacity scales with cluster size. Popularity replication in front, capacity partitioning behind: a pattern worth stealing (Wikitech CDN; techblog "road to ATS", 2020).
- **Cache the parse, not just the page.** Converting wikitext (with templates, Lua modules, hundreds of languages) into HTML is the expensive step — historically the main CPU cost. ParserCache stores rendered HTML for weeks, keyed by page+options, in its own MariaDB cluster; even logged-in users (who bypass the CDN due to personalization) usually hit ParserCache instead of re-parsing. Layered caching by *cost of regeneration*, not just by URL.
- **Open operations as an institution.** Wikitech documents production architecture, incident reports (public postmortems), SLOs, Grafana dashboards, and even the datacenter switchover runbooks. It's the largest site on earth where you can read the actual on-call docs — making it the best free "real system" case study available (wikitech.wikimedia.org).
- **Frugality as a forcing function.** The whole operation — a top-10 global site — runs on total org spend of ~$189M/yr, with the actual internet-hosting line only ~$3.4M (FY2024–25); Alphabet's 2024 *capex* alone was ~$52.5B. The three-orders-of-magnitude gap is the punchiest budget comparison on the internet, and the architecture above is *why* it's possible.

## Key numbers
- 296 billion pageviews across Wikimedia projects in 2024 (~10,000/sec average); English Wikipedia ~130B (2024, Wikistats via Wikipedia:Statistics — https://en.wikipedia.org/wiki/Wikipedia:Statistics and https://stats.wikimedia.org/)
- ~1 billion unique devices reached in December 2024 (2024, Wikimedia Foundation — https://wikimediafoundation.org/news/2025/10/17/new-user-trends-on-wikipedia/)
- ~16 edits/sec across all Wikimedia projects, over a third by bots (2024, Wikipedia:Statistics — https://en.wikipedia.org/wiki/Wikipedia:Statistics)
- ~1,300 bare-metal servers run the whole thing; ~90% of requests served from cache at the edge (2019, "Infrastructure of Wikipedia" talk, 36C3 — https://media.ccc.de/v/36c3-10592-infrastructure_of_wikipedia)
- 7 sites: 2 application DCs (eqiad Ashburn, codfw Carrollton) + 5 caching PoPs (Amsterdam, San Francisco, Singapore, Marseille 2022, São Paulo 2024) (accessed 2026, Meta "Wikimedia servers" — https://meta.wikimedia.org/wiki/Wikimedia_servers)
- Cache retention: Varnish frontend capped at 1 day; ATS backend max 24h; rendered page HTML kept up to ~14 days at app layer (accessed 2026, Wikitech CDN — https://wikitech.wikimedia.org/wiki/CDN)
- WMF FY2024–25 budget: $188.75M revenue and expenses (2024, WMF board resolution — https://foundation.wikimedia.org/wiki/Resolution:Approving_the_Wikimedia_Foundation_2024-2025_Annual_Plan_and_Budget); internet hosting expense line ~$3.4M (FY2024–25 audited financials — https://wikimediafoundation.org/about/financial-reports/); compare Alphabet 2024 capex ~$52.5B (2025, Alphabet Q4'24 earnings/10-K — https://abc.xyz/investor/)
- Historical baseline: ~350 servers at 3 sites, ~30,000 HTTP req/s peak, run by a handful of paid staff plus volunteers (2007, HighScalability "Wikimedia architecture" — https://highscalability.com/wikimedia-architecture/)
- Total datacenter energy: ~359 kW average draw, ~3.1 GWh/yr (2021, Meta "Wikimedia servers" — https://meta.wikimedia.org/wiki/Wikimedia_servers)

## Evolution timeline
- **2001** — One server running UseModWiki (Perl). Everything on one box.
- **2002–2003** — Custom PHP software ("Phase II/III") becomes **MediaWiki**; MySQL replication splits reads from writes.
- **2004–2007** — Squid HTTP caches added in front; grows to ~350 servers across 3 continents handling ~30k req/s peak (HighScalability, 2007) — the classic "LAMP + caching" era.
- **2013** — MySQL → **MariaDB** across production.
- **2014–2015** — PHP interpreter swapped for Facebook's **HHVM**; median page-save time roughly halved (WMF blog, 2014).
- **2017–2019** — HHVM abandoned upstream; migration back to **PHP 7** completed 2019.
- **2018–2020** — Backend cache layer migrated **Varnish → Apache Traffic Server**, simplifying the CDN and easing DC switchovers (techblog, 2020).
- **2022** — **Multi-DC active-active MediaWiki**: codfw serves live read traffic alongside eqiad; regular switchover drills become routine (Wikitech Multi-DC, 2022). Marseille PoP opens.
- **2023–2024** — MediaWiki serving fully containerized on **Kubernetes**; São Paulo PoP (magru) opens (2024) — first edge site in Latin America.

## Visualization hooks
- **The funnel.** A wide river of 10,000 req/s entering at the CDN, ~90% peeled off at Varnish/ATS, another slice at ParserCache/memcached, a trickle of tens of req/s of actual DB writes at the bottom. Label each layer with its "requests surviving" count.
- **Purge ripple.** World map: someone fixes vandalism in Ashburn; animate purge messages racing to all 7 sites and the old cached page winking out globally in ~seconds — contrasted with a TTL-only CDN where the vandalism lingers for a day.
- **Two cache physics.** Varnish as "every node holds the hits" (same hot articles glowing on every server) vs ATS as "each node owns a slice" (URL space cut into arcs, one copy each). Popularity-replication vs capacity-partitioning side by side.
- **1000:1.** A single edit pencil vs a wall of 600–1000 eyeball icons — the read:write ratio as a literal picture, annotated with the 2024 numbers.
- **Budget bar chart with a log axis.** Wikipedia hosting $3.4M vs WMF total $189M vs Alphabet capex $52.5B (all 2024-25) — the log axis itself becomes the joke.
- **Template edit blast radius.** Editing one popular infobox template → job queue fans out → millions of pages queued for lazy re-parse. Show the queue draining over hours rather than a synchronous stampede.
- **Switchover drill.** Two datacenters as twin control rooms; animate the "primary" crown moving from eqiad to codfw while reads continue uninterrupted from both — disaster recovery practiced as routine.
- **"You can read the runbook."** A mock screenshot-style panel of wikitech pages (CDN, incident reports, Grafana) with the caption: the only top-10 site whose ops docs are a public wiki.

## Sources
- Wikimedia CDN — Wikitech, continuously updated (accessed 2026) — HAProxy/Varnish/ATS layers, routing, TTLs, purge behavior, cache_text vs cache_upload. https://wikitech.wikimedia.org/wiki/CDN (primary, official ops docs)
- "Wikimedia's CDN: the road to ATS" — Wikimedia techblog, 2020 — why Varnish backend was replaced with Apache Traffic Server; architecture simplification. https://techblog.wikimedia.org/2020/11/25/wikimedias-cdn-the-road-to-ats/ (primary, official engineering blog)
- Wikimedia servers — Meta-Wiki, continuously updated (accessed 2026) — full stack list (gdnsd, LVS/PyBal, Varnish/ATS, MariaDB, Swift, memcached), DC locations, energy figures. https://meta.wikimedia.org/wiki/Wikimedia_servers (primary, official)
- Performance/Multi-DC MediaWiki — Wikitech, 2022 — the active-active design: reads from both DCs, write routing, session store choice (Cassandra/Kask). https://wikitech.wikimedia.org/wiki/Performance/Multi-DC_MediaWiki (primary, official ops docs)
- Switch Datacenter — Wikitech (accessed 2026) — the actual switchover runbook; evidence that failover is drilled routinely. https://wikitech.wikimedia.org/wiki/Switch_Datacenter (primary, official runbook)
- MediaWiki at WMF — Wikitech (accessed 2026) — how the PHP monolith is deployed (Kubernetes, app-layer caches). https://wikitech.wikimedia.org/wiki/MediaWiki_at_WMF (primary, official ops docs)
- "Infrastructure of Wikipedia" — 36C3 talk (Chaos Communication Congress), 2019 — ~1,300 bare-metal servers, ~90% cache hit rate, ops culture. https://media.ccc.de/v/36c3-10592-infrastructure_of_wikipedia (primary, talk by WMF SRE)
- Wikipedia:Statistics — English Wikipedia (accessed 2026, figures for 2024) — pageviews, edit rates, bot share; links to Wikistats. https://en.wikipedia.org/wiki/Wikipedia:Statistics (secondary aggregation of primary Wikistats data)
- Wikistats — stats.wikimedia.org (accessed 2026) — canonical pageview/edit/unique-device dashboards per project. https://stats.wikimedia.org/ (primary, official data)
- "Wikimedia architecture" — HighScalability, 2007 — the classic historical snapshot: ~350 servers, ~30k req/s, LAMP+Squid era. https://highscalability.com/wikimedia-architecture/ (secondary, based on WMF presentations; historical)
- WMF FY2024–25 Annual Plan & Budget resolution — Wikimedia Foundation Governance Wiki, 2024 — $188.75M budget. https://foundation.wikimedia.org/wiki/Resolution:Approving_the_Wikimedia_Foundation_2024-2025_Annual_Plan_and_Budget (primary, official)
- WMF audited financial statements (FY2024–25) — wikimediafoundation.org financial reports — internet hosting expense line (~$3.4M). https://wikimediafoundation.org/about/financial-reports/ (primary, audited financials)
- "New user trends on Wikipedia" — Wikimedia Foundation, 2025 — ~1B unique devices (Dec 2024), traffic composition. https://wikimediafoundation.org/news/2025/10/17/new-user-trends-on-wikipedia/ (primary, official)
