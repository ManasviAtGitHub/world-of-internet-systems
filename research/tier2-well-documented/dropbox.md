# Dropbox

## One-line hook
A file is not a blob — it's a list of hashes. Content-addressing every 4MB block let Dropbox sync
only what changed, dedupe the world's duplicates, fetch from the laptop next to you, and eventually
walk exabytes out of Amazon's cloud into hardware it built itself.

## The core problem
Keep the same folder byte-identical across hundreds of millions of devices — laptops that sleep
mid-write, three different OS filesystem semantics, flaky networks — while storing exabytes durably
and cheaply. Two very different hard problems hide inside: (1) **sync correctness**: converging
concurrent edits, moves, and shares without ever corrupting or losing a file (a
distributed-consistency problem running partly on hardware Dropbox doesn't control), and (2)
**storage economics**: at ~500PB and growing, renting S3 stopped making sense, so Dropbox executed
one of the largest cloud-exit migrations in history and became one of a handful of companies
operating exabyte-scale blob storage.

## Architecture overview

**End-to-end journey of one file save:**

1. **Chunk & hash.** The desktop client watches the filesystem; on change it splits the file into
   blocks of up to **4MB**, hashes each with **SHA-256**. The ordered hash list — the "blocklist" —
   *is* the file's content identity.
2. **Diff.** The client compares the new blocklist against the last-synced state. Only blocks whose
   hashes are new need to move: edit one sentence of a 1GB file and you upload ~4MB, not 1GB.
   (rsync-style delta within changed blocks and compression shrink it further.)
3. **Commit metadata, then blocks.** Dropbox splits its backend into two planes: a **metadata
   plane** (file journal / namespaces in sharded MySQL — which files exist, whose, which blocklists)
   and a **block plane** (content-addressed block storage). The client uploads missing blocks to
   block storage, then commits the new file revision against the metadata service, which checks it
   has every referenced hash.
4. **Storage: Magic Pocket.** Blocks land in Magic Pocket, Dropbox's in-house blob store. Immutable
   ~4MB blocks are appended into **1GB buckets**; buckets are grouped into **volumes** laid out on
   **OSDs** (dumb, petabyte-scale storage machines); a sharded-MySQL **Block Index** maps hash →
   bucket, and a **Replication Table** maps bucket → volume → OSDs. Cells of ~50PB are the
   failure/scale domain; a soft-state **Master** per cell orchestrates repairs. Fresh writes are
   fully replicated across at least two geographic **zones** within about a second; colder data is
   later erasure-coded within zones for efficiency. Durability is engineered to "12 nines."
5. **Fan-out.** Notification servers long-poll every online client in the namespace; peers pull the
   new metadata, see which hashes they lack, and download those blocks — from Dropbox, or via **LAN
   sync** directly from a machine on the same network (UDP discovery on port 17500, then HTTPS block
   fetch from the peer, secured by per-namespace certificates).
6. **Local commit.** Each client's sync engine (**Nucleus**, in Rust since 2020) reconciles three
   trees — the **Remote** view, the **Local** disk view, and the **Synced** merge base — planning
   moves/writes so every observable state is valid, then updates its local database and the actual
   files.

**Component list (plain text):**
- Desktop client / Nucleus sync engine (Rust): watcher, hasher, planner, three-tree reconciliation
- Notification service (long-polling fan-out)
- Metadata plane: file journal + namespace services on sharded MySQL (Edgestore)
- Block service API in front of Magic Pocket
- Magic Pocket: zones → cells (~50PB) → OSDs; Block Index (sharded MySQL); Replication Table;
  per-cell Master; replication + erasure coding
- LAN sync P2P subsystem
- Edge network / PoPs for user proximity (post-2016 buildout)

## Signature ideas

**Content-addressed block sync.** Naming data by its SHA-256 makes deduplication, delta sync,
resumable transfer, and integrity checking all fall out of one decision: identical blocks are stored
once regardless of who uploads them, and "what changed?" reduces to set-difference on hashes. This
2007-era design choice is the ancestor of everything else Dropbox built. (Documented in "Streaming
File Synchronization," 2014, and "Inside LAN Sync," 2015.)

**Streaming pipelined sync.** Rather than upload-then-notify-then-download, Dropbox pipelines: as
soon as some blocks of a large file are uploaded, receiving clients can begin fetching them, cutting
sync latency for big files dramatically ("Streaming File Synchronization," 2014).

**Magic Pocket — the great cloud exit.** From 2013–2016 Dropbox built its own exabyte-class blob
store and migrated **90%+ of ~500PB** off S3 in about 2.5 years — moving data at sustained rates
faster than most companies' total traffic, verifying every byte, while users noticed nothing. SEC
filings before the 2018 IPO showed ~$75M in infrastructure cost reduction over two years. The bet
only worked because Dropbox's workload (immutable 4MB blocks, write-once read-rarely) let them
specialize hardware and software together.

**SMR drives and hardware-software co-design.** In 2018 Dropbox became the first major tech company
to deploy shingled magnetic recording disks at petabyte scale. SMR drives are append-friendly but
hostile to random writes — a dealbreaker for general storage, but a perfect match for Magic Pocket's
immutable, sequential bucket writes. Owning the full stack let them ride the cheapest $/GB curve
(~100 disks per storage machine, hundreds of PB added).

**Nucleus: rewrite the sync engine in Rust.** Sync Engine Classic (Python, 2007 lineage) had
unstable file identifiers and weak consistency guarantees baked into its data model — un-fixable
incrementally. Nucleus (shipped 2020) redesigned the protocol around globally unique file IDs and
strong consistency (atomic moves, no observable duplicate/missing states), runs almost all logic on
**one control thread** for determinism, and leans on Rust's type system to make illegal states
unrepresentable.

**Deterministic simulation testing.** Nucleus's control-thread design enables randomized, fully
deterministic simulation: millions of generated scenarios per day (random filesystem events, network
partitions, crashes), each reproducible from an RNG seed. This testing style — pioneered publicly by
FoundationDB — is the reason a team of ~a dozen could safely swap the engine under hundreds of
millions of users.

**LAN sync.** If a colleague on your office network already has the block, fetch it from them: UDP
broadcast discovery, per-namespace TLS certificates distributed by Dropbox, HTTPS block fetch from
the fastest peer. Saves internet bandwidth for both user and Dropbox and makes office-wide sync feel
instant (2015 post; feature dates to ~2011).

## Key numbers
- **500+ PB of user data; 90% served from Magic Pocket after a ~2.5-year build+migration** —
  "Scaling to exabytes and beyond," dropbox.tech, Mar 14, 2016.
  https://dropbox.tech/infrastructure/magic-pocket-infrastructure
- **Durability designed for >99.9999999999% (12 nines) annually; availability >99.99%; cross-zone
  replication within ~1 second; cells ~50PB** — "Inside the Magic Pocket," James Cowling,
  dropbox.tech, May 6, 2016. https://dropbox.tech/infrastructure/inside-the-magic-pocket
- **~$74.6M reduction in infrastructure spend over the two years post-migration** — Dropbox S-1
  filing, 2018 (reported by GeekWire/Datacenter Dynamics).
  https://www.datacenterdynamics.com/en/analysis/how-dropbox-pulled-off-its-hybrid-cloud-transition/
  *(secondary reporting of primary SEC data)*
- **First petabyte-scale SMR deployment; ~100 disks per storage machine; hundreds of PB of SMR
  capacity being added** — "Extending Magic Pocket Innovation with the first petabyte scale SMR
  drive deployment," dropbox.tech, 2018.
  https://dropbox.tech/infrastructure/extending-magic-pocket-innovation-with-the-first-petabyte-scale-smr-drive-deployment
- **Exabyte+ scale, multi-region, >99.99% availability (2022 state)** — "Magic Pocket: Dropbox's
  Exabyte-Scale Blob Storage System," InfoQ article/QCon talk by Dropbox's Facundo Agriel,
  2022–2023. https://www.infoq.com/articles/dropbox-magic-pocket-exabyte-storage/
- **Hundreds of billions of files, trillions of revisions, exabytes of data, hundreds of millions of
  devices** in scope for the sync engine — "Rewriting the heart of our sync engine," Sujay Jayakar,
  dropbox.tech, Mar 9, 2020.
  https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine
- **Blocks up to 4MB, SHA-256 addressed; LAN discovery on UDP 17500** — "Inside LAN Sync," Matt Dee,
  dropbox.tech, Oct 13, 2015. https://dropbox.tech/infrastructure/inside-lan-sync
- **700M+ registered users, ~18M paying users (mid-2020s)** — Dropbox investor materials, aggregated
  at Backlinko/Expanded Ramblings, 2024–2026. https://backlinko.com/dropbox-users *(secondary
  compilation of primary filings)*

## Evolution timeline
- **2007–2008** — Launch: client-side 4MB chunking + SHA-256 from the start; metadata on Dropbox's
  own MySQL, blocks on Amazon S3, servers on EC2. The hybrid lasts eight years.
- **2011–2014** — Scale features on the same design: LAN sync, streaming/pipelined sync, massive
  dedup wins; user base passes 100M (2012) then 300M (2014).
- **2013–2015** — Magic Pocket built (roughly two years from prototype to production); migration
  machinery moves hundreds of PB; 90% of data in-house by Oct 2015.
- **2016** — Public reveal (Mar 2016): "Scaling to exabytes and beyond" + Wired's "Epic Exodus"
  feature; "Inside the Magic Pocket" architecture post (May); edge network / PoP buildout begins.
- **2018** — IPO filing quantifies ~$75M cost savings; first petabyte-scale SMR deployment; Magic
  Pocket crosses exabyte territory.
- **2020** — Nucleus ships: sync engine rewritten in Rust with deterministic simulation testing
  ("Rewriting the heart of our sync engine," March 2020); companion post "Testing sync at Dropbox."
- **2021–2023** — Magic Pocket matures (multi-region operations talks at QCon 2022); infrastructure
  story becomes the canonical "when to leave the cloud" case study.

## Visualization hooks
1. **A file becoming hashes** — a document sliced into 4MB bricks, each brick stamped with its
   SHA-256 fingerprint; edit one paragraph and exactly one brick glows "changed" and travels up the
   wire.
2. **Two planes** — metadata plane (tiny, hot, transactional: "what exists") and block plane (huge,
   cold, content-addressed: "the bytes") drawn as two parallel layers with the commit handshake
   between them.
3. **Magic Pocket nesting dolls** — block (4MB) inside bucket (1GB) inside volume inside OSD inside
   cell (50PB) inside zone (US-west/central/east), with replication arrows crossing zones in <1s.
4. **The exodus** — a convoy of data moving from the AWS cloud-castle to Dropbox's own datacenters:
   500PB, 2.5 years, 90% flag planted in Oct 2015; a $75M/2yr price tag falling away.
5. **Replicate-then-erasure-code** — fresh blocks shown as full copies (fast, safe), aging into
   striped parity fragments (cheap), the two regimes on a timeline.
6. **Three trees** — Nucleus's Remote/Local/Synced trees as three overlapping transparencies; sync =
   making all three identical; a conflict = where transparencies disagree.
7. **The simulator** — a dice-rolling machine feeding random chaos (crash! partition! rename storm!)
   into a boxed miniature of the sync engine, with a replay lever labeled "same seed = same bug."
8. **LAN shortcut** — two laptops in one office: the long route up to the cloud and back drawn
   faded; the short LAN hop drawn bold, with the UDP "anyone got block ab34…?" broadcast as a speech
   bubble.

## Sources
- **"Inside the Magic Pocket"** — dropbox.tech, James Cowling, May 6, 2016. The definitive
  architecture post: blocks/buckets/volumes, OSDs, Block Index, cells, zones, replication vs.
  erasure coding. *Primary.* https://dropbox.tech/infrastructure/inside-the-magic-pocket
- **"Scaling to exabytes and beyond"** — dropbox.tech, Akhil Gupta, Mar 14, 2016. The migration
  announcement: 500+PB, 90% in-house, why leave AWS. *Primary.*
  https://dropbox.tech/infrastructure/magic-pocket-infrastructure
- **"Rewriting the heart of our sync engine"** — dropbox.tech, Sujay Jayakar, Mar 9, 2020. Nucleus:
  why rewrite, why Rust, protocol redesign, control thread. *Primary.*
  https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine
- **"Testing sync at Dropbox"** — dropbox.tech, 2020. Deterministic simulation testing
  (CanopyCheck/Trinity-style randomized testing) for Nucleus. *Primary.*
  https://dropbox.tech/infrastructure/-testing-our-new-sync-engine
- **"Extending Magic Pocket Innovation with the first petabyte scale SMR drive deployment"** —
  dropbox.tech, 2018. SMR rationale and deployment scale. *Primary.*
  https://dropbox.tech/infrastructure/extending-magic-pocket-innovation-with-the-first-petabyte-scale-smr-drive-deployment
- **"Inside LAN Sync"** — dropbox.tech, Matt Dee, Oct 13, 2015. P2P discovery, per-namespace certs,
  block fetch. *Primary.* https://dropbox.tech/infrastructure/inside-lan-sync
- **"Streaming File Synchronization"** — dropbox.tech, 2014. Blocklists, pipelined upload/download
  of large files. *Primary.* https://dropbox.tech/infrastructure/streaming-file-synchronization
- **"Magic Pocket: Dropbox's Exabyte-Scale Blob Storage System"** — InfoQ article + QCon SF 2022
  talk, Facundo Agriel (Dropbox). Updated exabyte-era view. *Primary content via secondary venue.*
  https://www.infoq.com/articles/dropbox-magic-pocket-exabyte-storage/
- **"The Epic Story of Dropbox's Exodus from the Amazon Cloud Empire"** — Wired, Cade Metz, Mar
  2016. Narrative of the migration, team, and hardware program. *Secondary, deeply reported.*
  https://www.wired.com/2016/03/epic-story-dropboxs-exodus-amazon-cloud-empire/
- **"How Dropbox pulled off its hybrid cloud transition"** — Datacenter Dynamics, ~2018. S-1 cost
  figures ($74.6M/2yr), hybrid strategy (kept AWS for some international regions). *Secondary.*
  https://www.datacenterdynamics.com/en/analysis/how-dropbox-pulled-off-its-hybrid-cloud-transition/
- **Dropbox user stats** — Backlinko / Expanded Ramblings compilations of Dropbox filings,
  2024–2026. Registered vs. paying users. *Secondary compilations; check against latest 10-K for
  precision.* https://backlinko.com/dropbox-users
