# Ticketmaster & Booking Systems

> **Sourcing note:** Ticketmaster publishes far less engineering detail than Stripe/Shopify/Dropbox.
  This file mixes **[DISCLOSED]** facts (press releases, conference talks, interviews) with
  **[STANDARD PATTERN]** reservation-system design that any such system uses but Ticketmaster has
  not itself documented publicly. Each claim is tagged.

## One-line hook
Selling seats is the purest concurrency problem on the internet: 14 million people, 2 million
tickets, every seat unique, and every double-sell a lawsuit — solved with waiting rooms, TTL holds,
and (astonishingly) a 1970s VAX assembly engine still beating at the core.

## The core problem
Ticket inventory is the opposite of normal e-commerce stock. Every unit (seat 14, row F) is distinct
and non-fungible; demand outstrips supply by 10–100x; and the entire sale happens in minutes, not
weeks — an intentional thundering herd. The system must guarantee a seat is never sold twice
(correctness under fierce write contention), feel fair to humans while under industrial-scale bot
attack, and survive load that arrives as a vertical wall at exactly 10:00:00 AM. The Taylor Swift
Eras onsale (Nov 15, 2022) made the numbers public: 3.5 billion system requests in a day — 4x
Ticketmaster's previous peak — against roughly 2 million sellable tickets.

## Architecture overview

**End-to-end journey of one fan buying one seat (composite of disclosed pieces + standard design):**

1. **Pre-registration (days before) [DISCLOSED].** Verified Fan: fans register in advance;
   Ticketmaster scores registrations to filter bots/scalpers and sends invite codes to a subset
   sized to expected inventory (Eras: 1.5M coded fans of 3.5M registered; 2M waitlisted). This is
   *demand shaping before the sale* — the queue's first line of defense happens days early.
2. **Waiting room (T–30 min) [DISCLOSED].** Fans with codes join a virtual waiting room. At sale
   time, everyone present is assigned a **random** queue position (arriving 20 min early ≠
   advantage), converting a latency race that bots always win into a lottery, then a strict FIFO
   queue.
3. **Throttled admission [DISCLOSED].** The queue admits shoppers at a controlled rate — on the
   order of a few hundred concurrent shoppers per event — so the inventory system sees bounded
   concurrency no matter how many millions wait. Everyone else holds a queue token and sees live
   position updates.
4. **Seat selection [DISCLOSED mechanics, STANDARD implementation].** Admitted fans see an
   interactive seat map. Choosing seats places a **hold** — Ticketmaster's own guide states your
   selection is held for ~10 minutes while you check out. [STANDARD PATTERN]: a hold is a row/lock
   with a TTL — seat state machine `AVAILABLE → HELD(expires_at) → SOLD`, with expiry sweeping
   abandoned holds back to available. Implementations use pessimistic row locks, or an optimistic
   version check on the seat record, or a hold table / Redis key with TTL; the invariant is one
   active hold per seat.
5. **Checkout & payment [STANDARD].** Payment authorization runs while the hold is live; on success
   the hold converts atomically to SOLD; on failure/timeout the seat is released. The payment call
   is the classic "slow external dependency inside a lock window" problem — hence short TTLs.
6. **The inventory core [DISCLOSED].** The system of record for seat inventory is Ticketmaster's
   original "Host" ticketing engine — written largely in VAX assembly on a custom OS from the late
   1970s, today running on software emulators, wrapped by Java/API layers and, since ~2016, fronted
   by cloud-native services on Kubernetes/AWS. Modern services abstract it; the hard
   mutual-exclusion problem still lands on a lineal descendant of 1970s code.
7. **Bot defense throughout [DISCLOSED existence, undisclosed detail].** Edge-layer detection,
   behavioral flags (e.g., rapid refreshing can eject you as a suspected bot), CAPTCHAs, and
   Verified Fan scoring. In the Eras onsale, Ticketmaster attributed the overload substantially to
   bot attacks plus uninvited traffic.

**Component list (plain text):**
- Verified Fan registration + scoring pipeline (pre-sale demand filter) [DISCLOSED]
- Virtual waiting room + Smart Queue (random assignment → FIFO, throttled admission, position
  updates) [DISCLOSED]
- Bot detection at edge and in queue behavior [DISCLOSED existence]
- Interactive seat map service (real-time availability views) [DISCLOSED]
- Seat inventory engine: legacy VAX "Host" system under emulation + API wrappers [DISCLOSED]
- Hold manager with TTL; payment integration; order pipeline [~10-min hold DISCLOSED; mechanics
  STANDARD]
- Cloud platform: AWS + Kubernetes, 22,000+ VMs pre-migration, 7 datacenters [DISCLOSED, 2016-era]

## Signature ideas

**The virtual waiting room [DISCLOSED + industry pattern].** Decouple "absorbing demand" from
"serving demand": a lightweight, massively scalable queue tier soaks up the spike, while the fragile
inventory tier sees a fixed trickle. Randomizing positions at sale-open (rather than first-come)
removes the incentive to hammer at T-0 and neutralizes pure-speed bots. Vendors like Queue-it and
Cloudflare Waiting Room productized the same pattern; it's now standard for any onsale/drop.

**Holds with TTL — the reservation pattern [STANDARD, with the 10-min hold DISCLOSED].** The
universal answer to "lock inventory while a human decides": grant an exclusive, expiring claim. Too
short and checkouts fail; too long and abandoned carts strangle supply. The same pattern appears in
airline seat maps (PNR holds), hotel booking, and limited-edition drops. Key teaching contrast:
pessimistic locking (hold up front, guaranteed checkout) vs. optimistic (check at commit, risk "seat
taken" at the last second) — ticketing overwhelmingly chooses pessimistic-with-TTL because the item
is unique.

**Demand shaping before the sale [DISCLOSED].** Verified Fan moves the fight from milliseconds at
onsale to days beforehand: registration, identity scoring, invite codes sized to inventory.
Ticketmaster's stated baseline: historically ~40% of invited registrants buy, ~3 tickets each — so
1.5M codes ≈ sized to ~2M tickets. The Eras failure mode was that *uninvited* traffic and bots
showed up anyway, 4x their previous record.

**The 45-year-old core [DISCLOSED].** Ticketmaster's founding engineering (1976–1982) built its own
operating system and ticketing engine in VAX assembly for raw transaction speed — arguably the
original high-frequency inventory system. Rather than rewrite it, generations of architecture
wrapped it: emulators replaced hardware, Java APIs abstracted it, Kubernetes deploys around it. A
superb case study in strangler-fig modernization and in how correctness-critical cores outlive
everything around them.

**Overload disclosure as post-mortem [DISCLOSED].** The Eras press release is a rare public incident
report from ticketing: 3.5B requests (4x prior peak), 15% of interactions erroring, slowdowns forced
by pushing queues, and the memorable demand framing that Swift would need 900+ stadium shows to
satisfy the traffic. The follow-on Jan 2023 Senate hearing put more on the record (bot attack scale,
apology, capacity claims).

## Key numbers
- **3.5 billion total system requests** during the Eras onsale, **4x previous peak** — Ticketmaster
  press release, Nov 19, 2022.
  https://business.ticketmaster.com/press-release/taylor-swift-the-eras-tour-onsale-explained/
- **3.5M Verified Fan registrations (largest ever); 1.5M invite codes sent; 2M waitlisted; 2M
  tickets sold Nov 15, 2022 (single-day artist record); 2.4M total; ~15% of interactions had issues;
  <5% of tickets hit resale** — same press release, Nov 19, 2022.
- **~10-minute seat hold during checkout; only a few hundred concurrent shoppers admitted per sale;
  random queue assignment at sale open** — Ticketmaster blog "How the Ticketmaster Queue Works,"
  updated Oct 2025. https://blog.ticketmaster.com/how-ticketmaster-queue-works/
- **22,000+ VMs, 7 global datacenters, 21 ticketing systems, 250+ products, 65 product teams** at
  the start of the Kubernetes/AWS migration — Linux.com/CNCF case coverage, Dec 21, 2016.
  https://www.linux.com/news/ticketmaster-chooses-kubernetes-stay-ahead-competition/ (companion:
  USENIX LISA17 talk "VAX to K8s," 2017:
  https://www.usenix.org/sites/default/files/conference/protected-files/lisa17_slides_osborn.pdf)
- **~500M tickets/year across 30+ countries** — Ticketmaster marketing figure, repeated in
  aggregator stats pages (e.g., https://expandedramblings.com/index.php/ticketmaster/, 2024-era).
  *Secondary; treat as order-of-magnitude.* Live Nation 10-K reports fee-bearing tickets separately
  (~330–350M/year, mid-2020s; see Music Business Worldwide data page, 2025:
  https://www.musicbusinessworldwide.com/data/live-nation-total-tickets-sold-by-ticketmaster-annually/)
- **Core ticketing engine: VAX assembly on a custom OS, late 1970s origin, now emulated** — LISA17
  talk (2017) + Computer Weekly DevOps coverage (2017).
  https://www.computerweekly.com/news/450416287/How-to-apply-DevOps-practices-to-legacy-IT

## Evolution timeline
- **1976–1982** — Ticketmaster founded (Phoenix, 1976); builds its own VAX-based OS and
  assembly-language ticketing engine optimized for transaction throughput; wins venue contracts
  against Ticketron on speed.
- **1990s** — National phone + retail outlet network on the same Host systems; 1995-ish: web sales
  begin bolting HTTP frontends onto the Host.
- **2000s** — Java/API layers wrap the Host; interactive seat maps; hardware VAXes replaced by
  software emulation; scale via 7 datacenters, tens of thousands of VMs.
- **2014–2017** — Verified Fan and Smart Queue introduced (bot crisis era; US BOTS Act passes 2016);
  migration to AWS + Kubernetes begins ("let the makers make"), one of the earliest big K8s
  enterprise adoptions.
- **Nov 2022** — Eras Tour onsale meltdown: 3.5B requests, general sale canceled; rare public
  technical disclosure.
- **Jan 2023** — US Senate Judiciary hearing; Ticketmaster attributes failure to bot attack +
  unprecedented demand; antitrust scrutiny follows (DOJ suit vs. Live Nation, 2024).

## Visualization hooks
1. **The funnel of fairness** — 14M hopefuls → 3.5M registered → 1.5M coded → few hundred shopping
   at once → 1 seat each: a literal funnel with each narrowing labeled by mechanism (scoring,
   lottery, throttle, hold).
2. **Random shuffle at T-0** — waiting room as a crowd; at sale open, a hand shuffles them into a
   numbered line — visualizing why arriving early doesn't matter and why bots lose their speed
   advantage.
3. **Seat state machine** — one seat cycling `AVAILABLE → HELD (10:00 countdown) → SOLD` / `→
   expired → AVAILABLE`, with two buyers racing and exactly one winning the hold.
4. **The 3.5B wall** — request-rate timeline for Nov 15, 2022: normal peak line, 4x spike, annotated
   with bot share and the "15% of interactions" error band.
5. **Supply vs. demand absurdity** — 52 shows vs. the "900 stadium shows / 2.5 years of nightly
   concerts" quote drawn as calendar pages.
6. **Archaeology cross-section** — the system as geological strata: 1970s VAX assembly at the
   bedrock, emulator layer, Java API layer, Kubernetes/cloud topsoil — with a modern request
   drilling down through all of it.
7. **Pessimistic vs. optimistic locking, staged as a duel** — two checkout flows for the same seat:
   hold-first (one shopper sad early) vs. commit-time check (one shopper sad late, cart in hand).
8. **Bots vs. humans at the gate** — queue entrance with bouncer (Verified Fan scoring) turning away
   robot costumes; some slip through anyway (the Eras admission).

## Sources
- **"Taylor Swift | The Eras Tour Onsale Explained"** — Ticketmaster Business press release, Nov 19,
  2022. All the disclosed Eras numbers: 3.5B requests, registrations, codes, tickets sold, resale %.
  *Primary (company statement; self-serving framing).*
  https://business.ticketmaster.com/press-release/taylor-swift-the-eras-tour-onsale-explained/
- **"How the Ticketmaster Queue Works"** — Ticketmaster blog, updated Oct 2, 2025. Waiting room vs.
  queue, random assignment, throttled admission, 10-min hold, bot flags. *Primary (consumer-facing,
  but the only official queue mechanics doc).*
  https://blog.ticketmaster.com/how-ticketmaster-queue-works/
- **"VAX to K8s: Ticketmaster's Transformation to Cloud Native DevOps"** — USENIX LISA17 slides,
  2017. The VAX assembly Host story, emulation, DevOps-around-legacy. *Primary (conference talk by
  Ticketmaster).*
  https://www.usenix.org/sites/default/files/conference/protected-files/lisa17_slides_osborn.pdf
- **"Ticketmaster Chooses Kubernetes to Stay Ahead of Competition"** — Linux.com (CNCF case study
  coverage), Dec 21, 2016. 22k VMs, 7 DCs, 250 products, quotes from SVP Justin Dean. *Secondary
  reporting of primary interviews.*
  https://www.linux.com/news/ticketmaster-chooses-kubernetes-stay-ahead-competition/
- **"How to apply DevOps practices to legacy IT"** — Computer Weekly, 2017. Corroborates custom VAX
  OS + modern deployment of the emulated core. *Secondary.*
  https://www.computerweekly.com/news/450416287/How-to-apply-DevOps-practices-to-legacy-IT
- **Taylor Swift–Ticketmaster controversy** — Wikipedia (living article). Timeline, Senate hearing,
  DOJ suit; useful index to primary citations. *Tertiary; verify against its citations.*
  https://en.wikipedia.org/wiki/Taylor_Swift%E2%80%93Ticketmaster_controversy
- **Variety: "Ticketmaster Explains Taylor Swift Ticket Crisis"** — Variety, Nov 2022. Independent
  report on the same disclosures. *Secondary.*
  https://variety.com/2022/music/news/ticketmaster-explains-taylor-swift-ticket-crisis-eras-tour-1235435673/
- **Queue-it "Virtual Waiting Rooms: Everything You Need to Know"** — Queue-it (vendor), current.
  Best public write-up of general waiting-room design (the pattern, not Ticketmaster's
  implementation). *Secondary/vendor; use for STANDARD PATTERN material.*
  https://queue-it.com/blog/virtual-waiting-room/
- **Live Nation ticket volumes** — Music Business Worldwide data page (from Live Nation filings),
  2025. Fee-bearing ticket counts per year. *Secondary compilation of primary SEC data.*
  https://www.musicbusinessworldwide.com/data/live-nation-total-tickets-sold-by-ticketmaster-annually/
- **Reservation-pattern literature** — e.g., Alex Xu's *System Design Interview vol. 2*
  hotel-reservation chapter and ByteByteGo/HighScalability ticketing design posts. Source for the
  [STANDARD PATTERN] lock/hold/TTL taxonomy; none describe Ticketmaster internals.
  *Secondary/educational.*
