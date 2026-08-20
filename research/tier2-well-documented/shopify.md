# Shopify

## One-line hook
The internet's biggest argument for NOT using microservices: one Ruby on Rails monolith, sliced into
isolated "pods" of shops, absorbing Black Friday traffic spikes of 280+ million requests per minute.

## The core problem
Shopify is a multi-tenant platform hosting millions of merchant stores on shared infrastructure —
and its defining workload is the **flash sale**. A single celebrity drop can aim a stampede at one
shop; Black Friday–Cyber Monday (BFCM) aims a stampede at *all* of them simultaneously, with
checkout writes (inventory, orders, payments) that cannot be served stale from a cache. The problem
is therefore threefold: scale writes horizontally, stop one hot shop from degrading its neighbors,
and keep a codebase that thousands of engineers can change safely — all while peak traffic is 10x+
normal and revenue-per-minute during the peak is measured in millions of dollars.

## Architecture overview

**End-to-end journey of one storefront request → checkout during a flash sale:**

1. **Edge.** The buyer hits Shopify's edge/CDN layer (peaked at 284M requests/minute at BFCM 2024).
   Static assets are served here; dynamic requests continue inward. Bot and abuse filtering happens
   at the edge, and hot sales can put buyers into throttled checkout queues.
2. **Sorting Hat.** A routing layer (yes, named after Harry Potter) inspects the request, determines
   which shop it belongs to, and stamps it with the pod that owns that shop. Every shop lives in
   exactly one pod.
3. **Pod.** A pod is a fully isolated slice of the platform: its own MySQL cluster (sharded by
   `shop_id`), Redis, and memcached. Stateless tiers — app servers, job workers, load balancers —
   are shared and autoscaled, but any given request talks to exactly one pod's datastores. A pod
   melting down strands only the shops on it.
4. **Read path: Storefront Renderer.** Storefront page views (the vast majority of traffic) are
   served by a purpose-built Ruby app, rewritten from scratch (2019–2020) outside the monolith, that
   loads the merchant's Liquid theme plus product/collection/inventory data and returns HTML ~4x
   faster than the old monolith path.
5. **Write path: Shopify Core.** Cart, checkout, orders, payments, admin — the big modular Rails
   monolith. Checkout decrements inventory and creates orders inside the shop's pod database;
   payment goes through Shopify Payments (built on Stripe) or other gateways; heavy work (emails,
   webhooks, fulfillment sync) is deferred to background jobs.
6. **Async tier.** Massive background-job and Kafka event infrastructure fan out post-purchase work,
   keeping the synchronous checkout path minimal. During BFCM 2024, Shopify's databases absorbed
   1.17 trillion writes over the weekend.

**Component list (plain text):**
- Edge/CDN + bot mitigation + checkout throttling queues
- Sorting Hat request router (shop → pod mapping)
- Pods: sharded MySQL (by shop_id) + Redis + memcached, one active + one recovery region each; Pod
  Mover for live pod relocation
- Shopify Core: modular Rails monolith (~components by business domain: orders, shipping, inventory,
  billing), boundaries enforced by Packwerk/Wedge
- Storefront Renderer: standalone read-optimized rendering app
- Background jobs + Kafka event pipeline
- Vitess-sharded MySQL for the Shop app backend (newer, separate system)
- Load-testing (Genghis) + game-day tooling; runs on Google Cloud across multiple regions

## Signature ideas

**Pod architecture (shop sharding as failure isolation, not just scale).** In 2015 Shopify hit the
wall on vertical MySQL scaling. Instead of plain sharding — where any shard failure still breaks the
whole platform a little — they grouped shops into fully isolated pods with their own datastores, so
a failure is a *pod* outage, not a platform outage. Pods pair an active and a recovery datacenter,
and the "Pod Mover" can relocate a pod between datacenters in minutes with no downtime.

**The modular monolith stance.** Shopify's 2019 "Deconstructing the Monolith" post is the most-cited
defense of staying monolithic: microservices would multiply deploy pipelines, add network latency
and partial failure to every call, and make cross-cutting refactors organizationally painful.
Instead they reorganized ~6,000 classes from Rails-style technical layers into business-domain
components and built tooling (Wedge, later the open-sourced **Packwerk**) that fails CI when one
component reaches into another's internals. Modularity without distribution.

**Sorting Hat.** A thin, rule-driven routing tier that makes multi-pod topology invisible to the
application: the app is written as if there's one database; the router picks which universe the
request executes in. It's the keystone that lets a monolith behave like a horizontally scaled
system.

**The Storefront Renderer rewrite.** Rather than microservice-ifying everything, Shopify carved out
exactly one thing — the read-heavy storefront rendering path — into a from-scratch Ruby app with
aggressive multi-layer caching (in-memory for hottest queries, key-value store for frequent ones).
Result: p75 response under ~45ms, ~4x faster than the monolith it replaced, migrated with zero
downtime by shadow-verifying responses against the old implementation.

**BFCM readiness as an engineering culture.** Preparation starts in March: capacity modeling, a
"What Could Go Wrong" risk census, chaos-engineering game days injecting real faults into
production-like systems, and five+ full-platform scale tests (largest in 2025: 200M requests/minute)
using their internal load-generation tool **Genghis**. BFCM is treated as a scheduled, rehearsed
launch, and Shopify publishes the resulting stats (and a real-time visualization globe) every year.

**Vitess for the next generation.** For the Shop app's Rails backend (a consumer app, not
shop-shardable), Shopify adopted Vitess (2023–2024): queries rewritten to carry sharding keys,
parallel CI against Vitess and vanilla MySQL, careful splitting of cross-keyspace transactions.
Alongside "shard balancing" tooling that moves shops between shards at terabyte scale with zero
downtime, this is their modern horizontal-MySQL story.

## Key numbers
- **BFCM 2024 (Nov 29–Dec 2, 2024):** $11.5B GMV (+24% YoY); peak **$4.6M in sales/minute**; 76M+
  consumers; peak **284M edge requests/minute** and **80M+ app-server requests/minute**; 1.19
  trillion edge requests; **10.5 trillion DB queries**, 1.17 trillion DB writes; 12TB/min data
  moved; 7TB/s of logs at peak — Shopify news + Shopify Engineering, Dec 2024.
  https://www.shopify.com/news/bfcm-data-2024
- **BFCM 2025 (Dec 2, 2025 press release):** record **$14.6B GMV** (+27% YoY); peak **$5.1M/minute**
  at 12:01pm EST Black Friday; 81M+ consumers; 94,900+ merchants had their best day ever — Shopify
  investors press release.
  https://www.shopify.com/investors/press-releases/shopify-merchants-achieve-record-breaking-146-billion-black
  ; https://www.shopify.com/news/bfcm-data-2025
- **BFCM 2023:** MySQL fleet peaked at **19M queries/second**; 99.999%+ uptime; 29.7PB served
  (~5TB/min) — Shopify Engineering on X, Nov 2023.
  https://x.com/ShopifyEng/status/1729500627347157493
- **Scale tests at 200M requests/minute** across three GCP regions, five major tests April–October —
  Shopify Engineering "How we prepare Shopify for BFCM," Nov 20, 2025.
  https://shopify.engineering/bfcm-readiness-2025
- **Storefront Renderer: <~45ms for 75% of requests, ~4x faster** than the legacy path — Shopify
  Engineering, Aug 2020.
  https://shopify.engineering/how-shopify-reduced-storefront-response-times-rewrite
- **~6,000 classes** reorganized in the componentization project (begun 2017) — Shopify Engineering
  "Deconstructing the Monolith," Feb 21, 2019.
  https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity
- **Pods since 2015; pod moves between datacenters in minutes** — Shopify Engineering "A Pods
  Architecture to Allow Shopify to Scale," Xavier Denis, 2018.
  https://shopify.engineering/a-pods-architecture-to-allow-shopify-to-scale

## Evolution timeline
- **2004–2006** — Tobi Lütke builds Snowdevil (snowboard shop) on early Rails, which becomes
  Shopify; the storefront rendering code from this era survives ~15 years inside the monolith.
- **2014–2015** — Vertical MySQL scaling exhausted; sharding by shop begins, evolving into the **pod
  architecture** (fully isolated per-pod datastores) with Sorting Hat routing.
- **2017** — Componentization project: monolith reorganized by business domain; Wedge tooling;
  multi-DC active/recovery pods and Pod Mover mature.
- **2019** — "Deconstructing the Monolith" publishes the modular-monolith doctrine; Packwerk follows
  (open-sourced 2020). Storefront Renderer rewrite begins (Jan 2019).
- **2020** — Storefront Renderer ships platform-wide (~4x faster reads); Shopify runs on
  Kubernetes/Google Cloud; BFCM stats become an annual publishing tradition (with the live BFCM
  globe).
- **2023–2024** — Vitess adopted for Shop app backend; shard balancing at terabyte scale; YJIT (Ruby
  JIT, heavily funded by Shopify) speeds production Ruby ~15%.
- **2024–2025** — BFCM peaks: 284M edge req/min (2024); $14.6B GMV weekend (2025); readiness program
  formalized as year-round with 200M req/min scale tests.

## Visualization hooks
1. **The Sorting Hat** — literally draw it: a request wearing a hat at a fork in the road, being
   sent to Pod 42 of N identical isolated pod-islands, each island holding its own
   MySQL/Redis/memcached.
2. **Flash-sale spike anatomy** — a traffic graph of one product drop: vertical wall of requests,
   with layers peeling off at each defense (edge cache → checkout queue → pod database), annotated
   with what absorbs what.
3. **Monolith vs. microservices, honestly drawn** — one big well-partitioned building with internal
   walls (Packwerk-enforced doors) vs. a city of tiny buildings connected by fragile bridges
   (network calls); label the bridges with latency and partial failure.
4. **BFCM by the minute** — $5.1M/minute peak as a money-firehose gauge; 284M req/min as a second
   gauge; same clock (12:01pm EST Black Friday).
5. **Pod failure blast radius** — grid of pods, one on fire, flames contained to its cell; contrast
   with a single-shard-death-spiral diagram of the pre-pod design.
6. **The rewrite that wasn't a microservice** — storefront read path being lifted out of the
   monolith as a single clean slice (Storefront Renderer), everything else staying put; before/after
   response-time bars (~180ms → ~45ms p75).
7. **Game day** — engineers deliberately cutting a cable in a war-room scene; a resiliency matrix as
   a battle map (services × failure scenarios × playbooks).
8. **A trillion writes** — 1.17T database writes over one weekend visualized as grains of sand
   filling an hourglass labeled BFCM.

## Sources
- **"A Pods Architecture to Allow Shopify to Scale"** — Shopify Engineering, Xavier Denis, 2018. Pod
  definition, Sorting Hat, Pod Mover, multi-DC strategy. *Primary.*
  https://shopify.engineering/a-pods-architecture-to-allow-shopify-to-scale
- **"Deconstructing the Monolith: Designing Software that Maximizes Developer Productivity"** —
  Shopify Engineering, Kirsten Westeinde, Feb 21, 2019. The modular-monolith argument;
  componentization; Wedge. *Primary.*
  https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity
- **"How Shopify Reduced Storefront Response Times with a Rewrite"** — Shopify Engineering, Aug
  2020. Storefront Renderer architecture, caching layers, verified zero-downtime migration, perf
  numbers. *Primary.*
  https://shopify.engineering/how-shopify-reduced-storefront-response-times-rewrite
- **"How we prepare Shopify for BFCM"** — Shopify Engineering, Nov 20, 2025. Genghis load testing,
  game days, scale tests at 200M rpm, resiliency matrix; restates BFCM 2024 infra stats. *Primary.*
  https://shopify.engineering/bfcm-readiness-2025
- **Shopify BFCM 2024 data page** — Shopify News, Dec 2, 2024. GMV, peak $/minute, consumers, infra
  stats. *Primary (company-reported).* https://www.shopify.com/news/bfcm-data-2024
- **Shopify BFCM 2025 press release** — Shopify Investors, Dec 2, 2025. $14.6B GMV, $5.1M/min peak.
  *Primary (company-reported).*
  https://www.shopify.com/investors/press-releases/shopify-merchants-achieve-record-breaking-146-billion-black
- **Shopify Engineering BFCM 2023 stats thread** — X (@ShopifyEng), Nov 2023. 19M MySQL QPS peak,
  99.999%+ uptime. *Primary (informal channel).* https://x.com/ShopifyEng/status/1729500627347157493
- **"Horizontally scaling the Rails backend of Shop app with Vitess"** — Shopify Engineering, Jan
  17, 2024. Vitess migration mechanics. *Primary.*
  https://shopify.engineering/horizontally-scaling-the-rails-backend-of-shop-app-with-vitess
- **"Shard Balancing: Moving Shops Confidently with Zero-Downtime at Terabyte-scale"** — Shopify
  Engineering. Live shop moves between shards. *Primary.*
  https://shopify.engineering/mysql-database-shard-balancing-terabyte-scale
- **"Inside Shopify's Modular Monolith"** — Milan Milanović interview with Oleksiy Kovyrin
  (Shopify), 2024. Confirms current pod/routing/Packwerk picture. *Secondary (interview with
  insider).* https://newsletter.techworld-with-milan.com/p/inside-shopifys-modular-monolith
- **"How Shopify Built Its Live Globe for Black Friday"** — Pragmatic Engineer, 2023. The BFCM
  visualization system itself. *Secondary.*
  https://newsletter.pragmaticengineer.com/p/shopify-black-friday
