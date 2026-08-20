# WhatsApp

## One-line hook
The most extreme efficiency story on the internet: ~50 engineers running a store-and-forward,
end-to-end encrypted message switch for nearly a billion people, by betting everything on Erlang,
FreeBSD, and ruthless minimalism.

## The core problem
Deliver short messages between any two phones on Earth — instantly when both are online, reliably
when one is not — over flaky mobile networks, at a scale of hundreds of millions of simultaneous TCP
connections and tens of billions of messages a day, with a team small enough to fit in one room.
Later (2016), add a second constraint: the server must be able to route every message while being
cryptographically unable to read any of them.

## Architecture overview
WhatsApp began in 2009 as a customized fork of ejabberd, the open-source Erlang XMPP server, which
was progressively rewritten into a fully custom Erlang system speaking a stripped-down,
binary-optimized XMPP-derived protocol over SSL sockets. The servers ran (pre-Facebook-migration) on
bare-metal FreeBSD machines — dual hex-core Westmere-class boxes with ~100GB RAM — with a
custom-patched BEAM (Erlang VM). By 2014 the fleet was roughly 550 servers across dual datacenters:
~150 chat servers (each terminating ~1M phone connections), ~250 multimedia servers, and database
nodes running Mnesia (Erlang's in-memory database) holding ~18 billion records in ~2TB of RAM across
16 partitions. Services were partitioned 2–32 ways with pg2 process groups for addressing and
primary/secondary node pairs for redundancy.

**The end-to-end journey of one message (post-2016, with E2E encryption):**
1. Alice's phone holds a long-lived Signal-protocol session with Bob. If none exists yet, her client
   fetches Bob's public "prekey bundle" (identity key, signed prekey, one-time prekey) from the
   WhatsApp server and derives a shared session — Bob doesn't even need to be online (asynchronous
   X3DH handshake).
2. Alice's client encrypts the message body with a per-message key from the Double Ratchet (AES-256
   + HMAC-SHA256). The server never holds any private keys.
3. The ciphertext travels over Alice's persistent, encrypted TCP connection to her chat server.
4. The chat server routes it toward Bob's chat server. If Bob is connected, it is pushed down his
   open socket immediately. If Bob is offline, the ciphertext is queued server-side — the classic
   **store-and-forward** design — with a write-back cache in front of disk (98% hit rate; half of
   all messages are picked up within 60 seconds).
5. Bob's phone reconnects, drains its queue, decrypts, and sends an ack; the server deletes the
   stored ciphertext and relays delivery status back to Alice (the double check marks).
6. Media never rides the message path: images/video/audio are uploaded once to HTTP blob servers;
   the message carries only an encrypted pointer + thumbnail, and each recipient downloads and
   decrypts it.
7. Group messages use Signal's "Sender Keys": the sender distributes a group Sender Key once via
   pairwise sessions, then encrypts each group message a single time — server-side fan-out without
   server-side plaintext.
8. Since multi-device (2021), Alice's client encrypts the message N times — once per registered
   device of Bob's (and of her own other devices) — a **client-fanout** model, since each device has
   its own identity key.

Components (plain-text list):
- Phone clients (7 platforms in the early years) with persistent SSL/TLS sockets
- Chat servers (Erlang, custom XMPP-derived protocol, ~1M connections each)
- Offline message queues + write-back cache (store-and-forward core)
- Mnesia database clusters (in-RAM, partitioned, primary/secondary pairs)
- Multimedia/blob HTTP servers (Yaws, lighttpd) for media upload/download
- pg2-based partition/routing layer across the Erlang cluster
- Signal-protocol key server (prekey bundle distribution; holds only public keys)
- Push notification bridges (APNs/GCM) to wake idle phones

## Signature ideas
- **Millions of connections per box.** WhatsApp treated connections-per-server as a headline metric:
  1M established TCP sessions on one machine in 2011, then over 2M in 2012 (2.8M in benchmarks), via
  FreeBSD and BEAM tuning, lock-contention hunting, and instrumentation down to the kernel. They
  later deliberately backed off to ~1M per server to keep headroom for failure spikes.
- **Erlang as the whole company strategy.** Process-per-connection concurrency, hot code loading
  (multiple live deploys per day with no downtime), and OTP fault tolerance let a ~10-person server
  team do both dev and ops — the famous "40 million users per engineer" ratio. They patched BEAM
  itself: multiple timer wheels, round-robin async file I/O threads, parallel mnesia_tm transaction
  managers, and custom gen_factory/gen_industry dispatch layers to feed 11,000 cores without
  bottlenecks.
- **Store-and-forward with no message history.** The server is a switch, not an archive: messages
  are queued only until delivered, then deleted. This single design decision keeps storage bounded,
  makes E2E encryption natural, and is why the double-check-mark protocol exists (delivery receipts
  flow back through the same path).
- **Signal protocol at billion scale (2016).** WhatsApp completed the largest E2E encryption
  deployment in history in April 2016, applying X3DH asynchronous key agreement, Double Ratchet
  forward secrecy, and Sender Keys for groups — by default, for every chat, call, and attachment.
- **Client-fanout multi-device (2021).** Instead of the phone acting as the single source of truth
  (the old web client just mirrored the phone), each of up to 4 companion devices got its own
  identity key and its own server connection; senders encrypt per-device and the server maps account
  → device list. App state sync between a user's devices is itself E2E encrypted.
- **Do less, relentlessly.** No ads platform, no feed, minimal features, one core flow. Nearly every
  scaling talk from the team credits the small surface area — plus buying bigger bare-metal boxes to
  keep node count low — as the real scaling technique.

## Key numbers
- 1 million established TCP connections on a single server — 2011 (WhatsApp blog,
  https://blog.whatsapp.com/on-e-millio-n)
- 2+ million TCP connections on one box (Xeon X5675, FreeBSD 8.2) — 2012 (WhatsApp blog "1 million
  is so 2011", https://blog.whatsapp.com/1-million-is-so-2011); 2.8M in benchmark — 2012 (Rick Reed,
  Erlang Factory SF 2012, https://vimeo.com/44312354)
- 18 billion messages on Dec 31, 2013; ~450M users, 32 engineers, >8,000 cores at acquisition — Feb
  2014 (HighScalability,
  https://highscalability.com/the-whatsapp-architecture-facebook-bought-for-19-billion/)
- 465M monthly users; 19B inbound + 40B outbound messages/day; 147M peak concurrent connections;
  342K msgs/sec in / 712K out at peak; 600M photos/day; ~550 servers; 11,000+ cores; Mnesia with 18B
  records in ~2TB RAM — 2014 (Rick Reed, "That's 'Billion' with a B", Erlang Factory SF 2014;
  writeup:
  https://highscalability.com/how-whatsapp-grew-to-nearly-500-million-users-11000-cores-an/; talk:
  https://www.infoq.com/presentations/whatsapp-scalability/)
- ~50 engineers serving 900M users — 2015 (Wired,
  https://www.wired.com/2015/09/whatsapp-serves-900-million-users-50-engineers/)
- E2E encryption completed for all users/platforms — April 2016 (WhatsApp Encryption Overview
  whitepaper, https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf; Signal blog,
  https://signal.org/blog/whatsapp-complete/)
- 2 billion users — Feb 2020 (WhatsApp blog,
  https://blog.whatsapp.com/two-billion-users-connecting-the-world-privately)
- ~100 billion messages/day — Oct 2020 (Zuckerberg, Facebook Q3 2020 earnings; TechCrunch,
  https://techcrunch.com/2020/10/29/whatsapp-is-now-delivering-roughly-100-billion-messages-a-day)
- Up to 4 companion devices per account, each with its own identity key — 2021 (Engineering at Meta,
  https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/)

## Evolution timeline
- **2009** — Launch; ejabberd (Erlang XMPP server) as the starting point; FreeBSD bare metal.
- **2011** — 1M connections on one server; heavy FreeBSD/BEAM tuning era begins.
- **2012** — 2M+ connections per server; Rick Reed's first Erlang Factory talk; ejabberd essentially
  rewritten into a custom Erlang system.
- **2013–2014** — Hundreds of millions of users; Mnesia at multi-terabyte RAM scale; partitioned
  service architecture (pg2, primary/secondary pairs); Facebook acquires WhatsApp for $19B (Feb
  2014).
- **2014–2016** — Signal protocol rollout, completed April 2016: default E2E encryption for 1B+
  users.
- **2015–2019** — Gradual migration off SoftLayer/FreeBSD onto Facebook's Linux-based
  infrastructure; team stays famously small (~50 engineers at 900M users, 2015).
- **2021** — Multi-device architecture: per-device identity keys, client-fanout encryption, phone no
  longer required to be online.
- **2020s** — 2B+ users, ~100B messages/day; encrypted backups, and continued client-fanout/E2E
  extensions.

## Visualization hooks
- **One message's journey:** animated path phone → chat server → (recipient online? push : queue) →
  recipient phone, with the two check marks appearing as ack packets flow backward — overlay a
  padlock showing where content is ciphertext (everywhere except the two phones).
- **The prekey handshake with an offline recipient:** Bob deposits prekeys "in a mailbox" at the
  server; Alice grabs a bundle and starts an encrypted session while Bob's phone is dark — a great
  way to draw X3DH without math.
- **2M connections in one box:** a single server rectangle with 2,000,000 hair-thin lines converging
  on it; compare against a typical web server of the era (~10K connections) as a scale bar.
- **Users-per-engineer chart:** WhatsApp (900M/50) vs contemporaries (Facebook, Twitter, Google
  headcounts, 2015) — log-scale bar chart of the famous ratio.
- **Store-and-forward as a physical mail sorting office:** messages as parcels that never get opened
  (sealed = E2E), held in per-user pigeonholes, destroyed after pickup.
- **Group message fan-out, two ways:** naive (encrypt N times, send N times) vs Sender Keys (encrypt
  once, server fans out sealed copies) — side-by-side arrow-count comparison.
- **Multi-device fanout (2021):** one typed message splitting into N differently-encrypted copies,
  one per device icon (phone, laptop, tablet), each with its own key color; contrast with the old
  "phone mirrors to web" tether.
- **Hot code loading:** a running conveyor belt (live traffic) where an engineer swaps a machine
  part without stopping the belt — Erlang's party trick, and why they shipped multiple times a day.

## Sources
- "The WhatsApp Architecture Facebook Bought For $19 Billion" — HighScalability, 2014 — the
  canonical synthesis of the Erlang/FreeBSD stack, ejabberd origins, store-and-forward design,
  connection benchmarks. Secondary but built directly on primary talks/posts.
  https://highscalability.com/the-whatsapp-architecture-facebook-bought-for-19-billion/
- "How WhatsApp Grew to Nearly 500 Million Users, 11,000 cores, and 70 Million Messages a Second" —
  HighScalability, 2014 — detailed writeup of Rick Reed's 2014 Erlang Factory talk: fleet sizes,
  Mnesia internals, BEAM patches, failure stories. Secondary (faithful talk notes).
  https://highscalability.com/how-whatsapp-grew-to-nearly-500-million-users-11000-cores-an/
- "That's 'Billion' with a B: Scaling to the next level at WhatsApp" — Rick Reed, Erlang Factory SF
  2014 (video via InfoQ) — primary source for all 2014 scale numbers and Erlang engineering detail.
  https://www.infoq.com/presentations/whatsapp-scalability/
- "Scaling to Millions of Simultaneous Connections" — Rick Reed, Erlang Factory SF 2012 (Vimeo +
  slides) — primary source for the 2M/2.8M connections-per-server work. https://vimeo.com/44312354
- "1 million is so 2011" — WhatsApp blog, 2012 — primary announcement of >2M TCP connections on one
  FreeBSD box, with hardware specs. https://blog.whatsapp.com/1-million-is-so-2011
- WhatsApp Encryption Overview (Technical Whitepaper) — WhatsApp, first published 2016, updated for
  multi-device — primary source for Signal protocol integration: prekeys, pairwise sessions, Sender
  Keys, per-device identity. https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf
- "WhatsApp's Signal Protocol integration is now complete" — Signal (Open Whisper Systems) blog,
  2016 — primary source for the April 2016 full-E2E milestone.
  https://signal.org/blog/whatsapp-complete/
- "How WhatsApp enables multi-device capability" — Engineering at Meta, 2021 — primary source for
  the 2021 multi-device redesign: per-device keys, client-fanout, Automatic Device Verification.
  https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/
- "Why WhatsApp Only Needs 50 Engineers for Its 900M Users" — Wired, 2015 — primary journalism for
  the 50-engineers story and the Erlang framing.
  https://www.wired.com/2015/09/whatsapp-serves-900-million-users-50-engineers/
- "Two Billion Users — Connecting the World Privately" — WhatsApp blog, 2020 — primary source for
  the 2B-user milestone. https://blog.whatsapp.com/two-billion-users-connecting-the-world-privately
- "WhatsApp is now delivering roughly 100 billion messages a day" — TechCrunch, 2020 — secondary
  report of Zuckerberg's Q3 2020 earnings statement.
  https://techcrunch.com/2020/10/29/whatsapp-is-now-delivering-roughly-100-billion-messages-a-day
