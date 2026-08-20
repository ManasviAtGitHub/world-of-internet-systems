# Slack

## One-line hook
How a "boring" PHP + MySQL webapp grew a real-time nervous system: edge caches that pre-load what
you're about to need (Flannel), channel servers that fan every message out worldwide in ~500ms, and
the slow-motion escape from per-workspace sharding via Vitess.

## The core problem
Slack must feel instant — messages, typing indicators, presence dots, unread badges — for
organizations ranging from 3-person startups to enterprises with hundreds of thousands of employees,
across tens of millions of simultaneous WebSockets. Its original architecture assumed "a workspace
fits on one shard and one message server," which the product itself then broke: giant customers made
single shards melt (boot payloads, reconnect storms, hot shards), and shared channels/Enterprise
Grid made data span workspaces. Slack's story is the migration of a workspace-centric monolith to
channel-centric, edge-cached, horizontally sharded infrastructure — without ever going down.

## Architecture overview
Slack has two planes. The **webapp plane**: a large PHP/Hack monolith serving the HTTP API, backed
by MySQL (originally master-master pairs sharded by workspace; today mostly Vitess-managed MySQL
sharded by more granular keys like channel/user), plus an asynchronous job queue (Kafka front, Redis
workers) for everything deferrable (notifications, link unfurls, search indexing, billing). The
**real-time plane**: Java services — Channel Servers, Gateway Servers, Admin Servers, Presence
Servers — plus the Flannel edge cache, delivering events over WebSockets.

**One message's journey (per Slack's 2023 "Real-time Messaging" post):**
1. The client maintains a WebSocket to a nearby edge: historically to Flannel/Gateway Servers in one
   of several geographic regions (Consul for discovery, Envoy for load balancing/TLS in front).
2. Sending a message is an HTTP POST to the **webapp** API, which authenticates, persists the
   message to the database tier, and enqueues async jobs (push notifications, unfurls) on the job
   queue.
3. The webapp hands the event to a stateless **Admin Server**, which uses consistent hashing on the
   channel ID to locate the one **Channel Server** responsible for that channel.
4. The Channel Server — stateful, in-memory, holding recent history per channel (at peak ~16M
   channels served per host) — fans the event out to every **Gateway Server** worldwide that has at
   least one subscriber to that channel.
5. Each Gateway Server, which holds users' channel subscriptions, pushes the event down the
   WebSockets of every connected member; clients render it. End-to-end, delivery lands across the
   world in roughly 500ms.
6. Ephemeral events (typing indicators) follow the same real-time path but skip persistence;
   presence changes flow through **Presence Servers** with clients subscribing only to visible
   users.
7. **Flannel**, the application-level edge cache, sits on the WebSocket path: on boot the client
   gets a minimal payload, then lazily queries Flannel (users, channels, bots) instead of hammering
   the origin; Flannel keeps its per-team cache fresh by consuming the same event stream, and
   just-in-time pushes objects (e.g., the profile of a user who was just @mentioned) before clients
   ask.
8. Resilience: Consistent Hash Ring Managers (CHARMs) watch Channel Server health and swap a failed
   host's ring slots to a replacement in under 20 seconds; clients that lose sockets reconnect
   through the edge without a boot-payload stampede.

Components (plain-text list):
- Clients (desktop/web/mobile) — HTTP for actions, WebSocket for events
- Envoy edge load balancers + Consul service discovery
- Flannel edge cache (Go, per-team consistent hashing, lazy query API)
- Gateway Servers (Java, stateful, per-user subscriptions, multi-region)
- Channel Servers (Java, stateful in-memory, consistent-hashed by channel ID)
- Admin Servers (Java, stateless webapp↔CS bridge), Presence Servers
- PHP/Hack webapp monolith (API, business logic)
- MySQL/Vitess datastore tier (vtgate routing, vttablet-managed shards)
- Job queue: Kafkagate (Go, HTTP→Kafka) → Kafka (durable buffer) → JQRelay (Go) → Redis → workers

## Signature ideas
- **Flannel: a cache that predicts you.** Slack's clients originally downloaded the whole team state
  at connect (`rtm.start`) — untenable once big customers had thousands of users, and catastrophic
  during reconnect storms. Flannel, deployed at edge POPs, gives clients a tiny boot payload plus an
  on-demand query API, uses consistent hashing so a team's users hit the same warm cache, and
  proactively pushes objects it can predict clients will need. It absorbed 4M+ simultaneous
  connections and ~600K queries/sec in 2017, cutting connect times ~30%+.
- **Channel-centric fan-out with consistent hashing.** The real-time core is a two-hop pub/sub:
  channels hash to Channel Servers; Gateway Servers subscribe on behalf of users. This decouples
  "who owns the channel" from "where the user is connected," lets each layer scale independently,
  and made cross-workspace shared channels possible (the older design sharded message servers by
  team, which shared channels broke — Slack re-sharded the messaging tier by channel/topic).
- **Workspace sharding and its escape hatch (Vitess).** The founding data model — each workspace
  pinned to a master-master MySQL pair — bought years of easy scaling and clean isolation but
  produced hot shards (one giant customer = one melting database), full outages for the customer
  whose shard died, and no home for cross-workspace data (Enterprise Grid, Slack Connect). From 2017
  to 2020 Slack moved ~99% of query load onto Vitess, re-sharding by finer keys, reaching 2.3M QPS
  (2M reads/300K writes) at 2ms median.
- **Kafka in front of Redis, not instead of it.** Slack's Redis job queue (1.4B jobs/day) had a
  fatal coupling: dequeuing required free Redis memory, so a backlog could wedge the whole system —
  which happened in a major incident. Rather than rewrite everything, they inserted Kafka as a
  durable buffer (Kafkagate → Kafka → JQRelay → Redis), keeping worker code untouched while making
  enqueue spikes survivable — a model lesson in incremental re-architecture.
- **Master-master MySQL with app-level conflict tolerance.** Early Slack ran statement-based
  master-master replication, accepting occasional conflicts in exchange for high write availability
  per workspace — a pragmatic, unfashionable choice defended in Keith Adams' "How Slack Works" talk,
  and the substrate everything else evolved on.
- **Eventful pragmatism: persist first, then fan out.** Unlike systems that treat the real-time pipe
  as the source of truth, Slack writes the message durably via the webapp before real-time delivery,
  so the WebSocket layer can be lossy/restartable and clients reconcile via history APIs — the
  reason CHARM-driven Channel Server replacement in <20s is safe.

## Key numbers
- Flannel: 4M+ simultaneous connections at peak, ~600K client queries/sec — 2017 (Slack Engineering,
  https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/)
- Flannel grown to 5M+ simultaneous connections, 1M+ queries/sec at peak; presence event traffic cut
  ~5x by pub/sub presence — 2017 (Bing Wei, "Scaling Slack", QCon SF 2017 via InfoQ,
  https://www.infoq.com/presentations/slack-scalability/)
- Job queue: 1.4B jobs/day, peak 33K jobs/sec; 16-broker Kafka cluster, 32 partitions/topic, 3x
  replication — 2017 (Slack Engineering, https://slack.engineering/scaling-slacks-job-queue/)
- Vitess migration: started 2017; ~99% of query load by Dec 2020; 2.3M QPS (2M reads, 300K writes),
  2ms median / 11ms p99 — 2020 (Slack Engineering,
  https://slack.engineering/scaling-datastores-at-slack-with-vitess/)
- Real-time plane: ~16M channels served per Channel Server host at peak; failed CS replaced in <20s;
  worldwide delivery ~500ms; tens of millions of concurrent WebSockets — 2023 (Slack Engineering,
  https://slack.engineering/real-time-messaging/)
- 10M+ daily active users — 2019 (Slack blog,
  https://slack.com/blog/news/slack-has-10-million-daily-active-users)

## Evolution timeline
- **2013–2015** — Founding architecture: PHP monolith, per-workspace master-master MySQL shards, one
  Java Message Server per team, `rtm.start` full-snapshot boot (documented in Keith Adams' QCon SF
  2016 talk "How Slack Works").
- **2016** — Big-customer growing pains: giant boot payloads, reconnect storms, hot shards; the
  Redis job-queue near-death incident motivates queue redesign.
- **2017** — Flannel deployed at edge POPs (lazy boot + cache); Kafka inserted in front of the Redis
  job queue; Vitess migration begins; messaging tier re-sharded from team-based to
  channel/topic-based to support shared channels.
- **2018–2019** — Enterprise Grid and Slack Connect (cross-workspace shared channels) cement the
  move away from workspace-locality assumptions; 10M DAU (2019).
- **2020** — Vitess reaches ~99% of query load; workspace-sharded MySQL effectively retired as the
  scaling model.
- **2021–2023** — Mature real-time architecture published: Channel/Gateway/Admin/Presence server
  split, CHARMs, Envoy + Consul, multi-region gateways ("Real-time Messaging," 2023).

## Visualization hooks
- **The two-plane diagram:** HTTP "action" path (client → webapp → DB → job queue) drawn in one
  color and the WebSocket "event" path (Admin → Channel Server → Gateways → clients) in another,
  meeting at the message-send moment — Slack's whole architecture in one picture.
- **Message world tour in 500ms:** animated single message hopping webapp → channel server → three
  continents' gateway servers → dozens of clients, with a running millisecond clock.
- **The boot payload diet:** old `rtm.start` as a truck delivering the entire office (every user,
  channel, emoji) vs. Flannel handing over a slim folder and answering questions on demand — with
  payload-size bars for a 10-person vs 100,000-person org.
- **Reconnect storm absorbed at the edge:** an office building's power flickers; thousands of
  clients reconnect and slam into the Flannel edge wall, which stays warm, while the origin barely
  notices — before/after traffic graphs.
- **Consistent hash ring with CHARM repair:** channels as beads on a ring distributed across Channel
  Servers; one server dies, its arc glows red, a CHARM slots in a replacement within a 20-second
  timer.
- **Hot shard problem:** rows of workspace shards as evenly sized boxes, one giant enterprise
  customer bulging out of its box — then the Vitess re-shard redistributing that customer across
  many shards (and the 2.3M QPS counter).
- **Kafka as the reservoir:** job flood hitting a dam (Kafka) that releases a steady flow into Redis
  turbines — annotate the old failure ("reservoir was the turbine; flood = blackout").
- **Fan-out cascade:** one Channel Server event splitting into per-region Gateway Server copies,
  then into thousands of WebSocket deliveries — a two-stage tree that visualizes why the middle tier
  exists at all.

## Sources
- "Real-time Messaging" — Slack Engineering, 2023 — the definitive published description of
  Channel/Gateway/Admin/Presence servers, message flow, CHARMs, and per-host scale. Primary.
  https://slack.engineering/real-time-messaging/
- "Flannel: An Application-Level Edge Cache to Make Slack Scale" — Slack Engineering, 2017 —
  boot-payload problem, edge cache design, consistent hashing by team, JIT push, scale stats.
  Primary. https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/
- "Scaling Slack's Job Queue" — Slack Engineering, 2017 — Redis failure modes, Kafkagate/JQRelay
  design, throughput numbers. Primary. https://slack.engineering/scaling-slacks-job-queue/
- "Scaling Datastores at Slack with Vitess" — Slack Engineering, 2020 — workspace-sharding limits,
  Vitess rationale, migration timeline and QPS/latency figures. Primary.
  https://slack.engineering/scaling-datastores-at-slack-with-vitess/
- "How Slack Works" — Keith Adams (Slack Chief Architect), QCon San Francisco 2016 — the classic
  pre-Flannel architecture: PHP monolith, master-master MySQL per workspace, per-team message
  server. Primary (talk). https://qconsf.com/sf2016/sf2016/presentation/how-slack-works.html
- Keith Adams on the Architecture of Slack — InfoQ Podcast, 2017 — companion interview covering
  MySQL master-master/SBR choices and edge caching direction. Primary (interview).
  https://www.infoq.com/podcasts/slack-keith-adams/
- "Scaling Slack" — Bing Wei, QCon SF 2017 (InfoQ video + transcript) — Flannel evolution, lazy
  loading, pub/sub presence, team→topic re-sharding for shared channels. Primary (talk).
  https://www.infoq.com/presentations/slack-scalability/
- "Slack has 10 million daily active users" — Slack blog, 2019 — DAU milestone. Primary (company
  announcement). https://slack.com/blog/news/slack-has-10-million-daily-active-users
- "How Slack Supports Billions of Daily Messages" — ByteByteGo newsletter, 2022 — clear secondary
  synthesis of the above; useful for diagram cross-checking, not as a numbers source. Secondary.
  https://blog.bytebytego.com/p/how-slack-supports-billions-of-daily
