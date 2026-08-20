# Discord

## One-line hook
A chat app where one "room" can hold a million live users: Discord's guild-process model (Elixir),
polyglot pragmatism (Rust where GC hurts), and the internet's most famous database migration
(Cassandra → ScyllaDB for trillions of messages).

## The core problem
Group chat fan-out is quadratic pain: every message, presence flicker, or typing event in a server
("guild") must be pushed in real time to every connected member — and Discord's biggest guilds grew
from 30,000 concurrent users (2017) to over 1,000,000 online (Midjourney, 2023). Simultaneously,
every message ever sent must be stored and instantly retrievable: billions, then trillions of rows
with wildly skewed access patterns (dead-quiet friend servers vs. firehose communities), plus
low-latency voice for millions of concurrent speakers.

## Architecture overview
Discord's real-time plane is an Elixir/Erlang cluster; its API/storage plane is Python (API
monolith) plus Rust data services over ScyllaDB; voice/video is a custom C++ WebRTC SFU fleet.

**Real-time (text/events) data flow — one message's journey:**
1. A client holds a persistent WebSocket to a **Gateway/session** node; connecting spawns an Elixir
   session process for that client.
2. The client sends a message via the API; the API layer persists it (through the Rust data services
   into ScyllaDB) and hands it to the real-time system.
3. Every guild lives as a **single Elixir GenServer process** on some node in the cluster — the
   routing point that knows all connected sessions for that guild.
4. The guild process fans the message event out to every member's session process — across nodes
   this uses **Manifold**, which batches sends per remote node instead of N direct sends — and each
   session pushes it down its WebSocket.
5. Offline members get push notifications; on reconnect, clients resync state and fetch missed
   messages from the message store by channel.
6. Reads of history go: client → API → Rust data service (request coalescing: concurrent identical
   queries collapse into one) → ScyllaDB partition keyed by `(channel_id, time bucket)`.

**Voice data flow:** the client's Gateway WebSocket negotiates voice state; the **Guilds** service
picks a healthy **Discord Voice** server (via etcd service discovery) in a nearby region; clients
then send Opus audio over encrypted UDP to that server's **SFU (Selective Forwarding Unit)**, which
forwards each speaker's stream to all other participants. No peer-to-peer — this scales to large
rooms and never reveals members' IP addresses to each other. On voice-server death or DDoS, clients
detect the dead connection and the gateway reassigns them to a new server.

Components (plain-text list):
- Clients (desktop/web/mobile) with one WebSocket (gateway) + UDP (voice)
- Elixir gateway/session nodes (process per connected client)
- Elixir guild nodes (process per guild = fan-out router); Manifold for inter-node batched sends
- Python API monolith (HTTP, message create/edit, permissions)
- Rust data services layer (request coalescing, consistent-hash routing by channel)
- ScyllaDB message cluster (formerly Cassandra, originally MongoDB)
- Discord Voice servers: C++ signaling + SFU, 13+ regions, etcd-based discovery
- Push notification + media/CDN subsystems

## Signature ideas
- **Process-per-guild in Elixir.** Each guild is one GenServer that serializes and routes all events
  for that community; sessions are also processes. This maps Erlang/OTP's actor model directly onto
  the product's social structure. The catch: work grows roughly quadratically with guild size (more
  members produce more events × more recipients), which drove years of optimization — Manifold
  (batch fan-out per node), FastGlobal (0.3µs shared constant lookups vs 7µs), Semaphore (ETS-based
  overload guards), and eventually the 2023 "Maxjourney" push (passive connections, member-list ring
  buffers, process splitting) for million-user guilds.
- **The database ladder: MongoDB → Cassandra → ScyllaDB.** MongoDB broke at ~100M messages in Nov
  2015 when data+index outgrew RAM. Cassandra (2016–2022) brought linear scale with `((channel_id,
  bucket), message_id)` partitioning and Snowflake IDs, but suffered hot partitions, tombstone
  disasters (a channel with millions of deleted messages could stall the cluster), and Java GC
  pauses. ScyllaDB (C++, shard-per-core, no GC) cut the messages cluster from 177 to 72 nodes and
  p99 reads from 40–125ms to 15ms.
- **Rust wherever GC is the enemy.** The Read States service (tracks what every user has read;
  touched on every connect/send/read) had 2-minute-interval latency spikes purely from Go's forced
  GC cycles over a huge LRU cache. Rewritten in Rust (ownership = free on eviction, no GC), it beat
  Go on every metric and let them grow the cache to 8M entries. Rust also powers the data services
  layer and the migration tooling.
- **Request coalescing in a data services tier.** Between API and database, Rust services collapse
  simultaneous identical reads ("everyone opens the same hot channel") into a single DB query, with
  consistent-hash routing by channel so the same service instance absorbs a channel's hotness — the
  shield that makes hot partitions survivable.
- **The 9-day trillion-message migration.** Dual-write to Cassandra + ScyllaDB, then a custom Rust
  migrator streamed historical data at up to 3.2M messages/sec, finishing trillions of rows in nine
  days (est. 3 months with the standard approach), with cutover in May 2022 and zero downtime. Also
  introduced "super-disks": GCP local SSDs RAID-mirrored with persistent disks for low-latency reads
  plus durable writes.
- **SFU voice at scale.** A custom C++ media engine on the WebRTC native library (same code across
  platforms), with the SFU forwarding selected streams, enforcing server-side mute/deafen by
  dropping packets, and bridging browser/native signaling differences — the reason Discord voice
  works for both a 3-friend call and a 1,000-listener stage.

## Key numbers
- 100 million stored messages when MongoDB's RAM limit hit — Nov 2015 (Discord blog, 2017,
  https://discord.com/blog/how-discord-stores-billions-of-messages)
- ~5 million concurrent users, millions of events/sec through the Elixir system; largest guild
  /r/Overwatch at ~30,000 concurrent; naive fan-out cost 30–70µs per `send/2` → up to ~2.1s per
  event — 2017 (Discord blog,
  https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
- 2.6M+ concurrent voice users; 850+ voice servers in 13 regions; 220+ Gbps egress; 120M packets/sec
  — Sept 2018 (Discord blog,
  https://discord.com/blog/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc)
- Read States: tens of millions of entries in cache; Go GC spikes every 2 minutes; Rust cache later
  raised to 8M entries with microsecond averages — Feb 2020 (Discord blog,
  https://discord.com/blog/why-discord-is-switching-from-go-to-rust)
- Trillions of messages; Cassandra cluster at 177 nodes (early 2022) → 72 ScyllaDB nodes; p99 reads
  40–125ms → 15ms; p99 writes 5–70ms → 5ms; migration at up to 3.2M msgs/sec in 9 days; cutover May
  2022 — March 2023 (Discord blog,
  https://discord.com/blog/how-discord-stores-trillions-of-messages)
- 1,000,000+ online users in a single guild (Midjourney); guild capacity expanded ~15x by the
  Maxjourney team — Oct 2023 (Discord blog,
  https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server;
  InfoQ summary 2024, https://www.infoq.com/news/2024/01/discord-midjourney-performance/)

## Evolution timeline
- **2015** — Launch. MongoDB single replica set for messages; Elixir chosen for the real-time
  gateway from day one.
- **Nov 2015 – Jan 2016** — Messages outgrow MongoDB RAM at ~100M rows; migration to Cassandra with
  time-bucketed channel partitions.
- **2016–2017** — Elixir scaling era: Manifold, FastGlobal, Semaphore; ~5M concurrent users;
  tombstone crisis and Cassandra tuning.
- **2018** — Custom WebRTC stack matures: C++ SFU, 13 regions, 2.6M concurrent voice users.
- **2019–2020** — Rust adoption: Read States rewritten from Go; Rust becomes a first-class backend
  language.
- **2020–2022** — Cassandra pain compounds (17 → 177 nodes, GC firefighting, hot partitions); Rust
  data services with request coalescing built in front of storage.
- **May 2022** — ScyllaDB cutover after 9-day dual-write migration of trillions of messages
  (published March 2023).
- **2023** — Maxjourney: million-online single guilds (Midjourney), ~15x guild capacity via
  profiling-driven Elixir work.

## Visualization hooks
- **The fan-out explosion:** one message entering a guild process and splitting into 30,000 (then
  1,000,000) arrows to session processes — animate the difference between naive per-process sends
  and Manifold's "one bundle per node, unpacked on arrival."
- **Process map as a city:** each guild a building whose single "mailroom" (GenServer) routes
  everything; small guilds are cottages, Midjourney is a skyscraper with a visibly overworked
  mailroom — makes the quadratic-growth problem tangible.
- **The database ladder timeline:** three eras (MongoDB / Cassandra / ScyllaDB) drawn as vessels of
  growing size, annotated with what broke each one (RAM ceiling / tombstones + GC / —), with the
  node-count drop 177→72 as the punchline.
- **Hot partition heatmap:** a grid of `(channel, bucket)` partitions where one community channel
  glows red while thousands of friend-server partitions stay cold — then show request coalescing
  collapsing a burst of identical reads into one DB arrow.
- **The 9-day migration race:** dual-write pipes into two databases side by side while a Rust "pump"
  backfills history at 3.2M msgs/sec; progress bar labeled "estimated: 3 months → actual: 9 days."
- **SFU vs P2P:** left panel a 10-person P2P mesh (45 links, IPs exposed); right panel 10 clients
  star-connected to one SFU (10 links, IPs hidden) — the cleanest possible voice-architecture
  diagram.
- **GC pause seismograph:** Read States latency as a seismogram with a tremor every 2 minutes (Go),
  flatlining after the Rust rewrite — a before/after strip chart.
- **Tombstone graveyard:** a channel drawn as a shelf where millions of "deleted" messages remain as
  gravestones the reader must walk past to find one live message — why reading an empty channel took
  down nodes.

## Sources
- "How Discord Stores Billions of Messages" — Discord blog (Stanislav Vishnevskiy era), 2017 —
  MongoDB→Cassandra migration, schema design, Snowflake IDs, tombstone crisis. Primary.
  https://discord.com/blog/how-discord-stores-billions-of-messages
- "How Discord Scaled Elixir to 5,000,000 Concurrent Users" — Discord blog (Stanislav Vishnevskiy,
  CTO), 2017 — guild/session process model, fan-out costs, Manifold/FastGlobal/Semaphore. Primary;
  the canonical Elixir-at-scale post.
  https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users
- "How Discord Handles Two and Half Million Concurrent Voice Users using WebRTC" — Discord blog,
  2018 — SFU architecture, gateway/guilds/voice service split, regions, failover, scale stats.
  Primary.
  https://discord.com/blog/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc
- "Why Discord is switching from Go to Rust" — Discord blog, 2020 — Read States service, Go GC spike
  analysis, Rust rewrite results. Primary; widely cited in the Go/Rust debate.
  https://discord.com/blog/why-discord-is-switching-from-go-to-rust
- "How Discord Stores Trillions of Messages" — Discord blog (Bo Ingram), 2023 — Cassandra pain,
  ScyllaDB migration, Rust data services, request coalescing, super-disks, 9-day migration. Primary;
  the famous one. https://discord.com/blog/how-discord-stores-trillions-of-messages
- "Maxjourney: Pushing Discord's Limits with a Million+ Online Users in a Single Server" — Discord
  blog, 2023 — profiling the guild process, quadratic scaling, 15x capacity work for Midjourney.
  Primary.
  https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server
- "Discord Scales to 1 Million+ Online MidJourney Users in a Single Server" — InfoQ, 2024 —
  independent summary/verification of the Maxjourney work. Secondary.
  https://www.infoq.com/news/2024/01/discord-midjourney-performance/
- "How Discord Migrated Trillions of Messages to ScyllaDB" — The New Stack, 2023 — interview-based
  retelling of the migration with extra operational color. Secondary.
  https://thenewstack.io/how-discord-migrated-trillions-of-messages-to-scylladb/
