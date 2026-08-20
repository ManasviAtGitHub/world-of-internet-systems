# DNS — The Internet's Phone Book

## One-line hook
Thirteen IP addresses, answered by more than 2,000 servers scattered across every continent, sit at the root of a distributed database that turns every name you type into a number — and when it breaks, the internet doesn't just slow down, it vanishes.

## The core problem
Humans remember names; routers forward packets to numeric addresses. Something has to map `example.com` → `93.184.216.34` for billions of devices, updated constantly, with no central bottleneck. A single master hosts file (which is literally how the early ARPANET did it — one file, hand-distributed) stopped scaling in the early 1980s. DNS (RFC 1034/1035, 1987) solved it with three moves: a **hierarchy** so no server needs to know everything, **delegation** so every organization runs its own slice, and **caching** so almost no query ever travels the full path.

## How it works

Two fundamentally different roles:
- **Recursive resolver** (the "librarian"): the server your device actually asks — your ISP's, or a public one like 1.1.1.1 / 8.8.8.8. It does the legwork and caches answers.
- **Authoritative servers** (the "books"): hold the actual records for a zone and answer only for it. Three tiers: root → TLD (.com, .org, .in) → the domain's own authoritative nameserver.

A fully-uncached lookup for `example.com` (~20-120 ms total):
1. Browser checks its own cache, then the OS cache/hosts file (~<1 ms). Miss.
2. Stub resolver sends the query to the configured recursive resolver.
3. Resolver → a **root server**: "where's .com?" Root replies with the .com TLD server list (roots are so heavily cached this step is often skipped in practice).
4. Resolver → a **.com TLD server**: "where's example.com?" Reply: the domain's authoritative nameservers (the NS records set when the domain was registered).
5. Resolver → the **authoritative server**: "A record for example.com?" Reply: the IP, plus a TTL.
6. Resolver caches the answer for TTL seconds and returns it to the stub; browser opens the TCP/QUIC connection.

Each referral is one round trip, which is why an empty-cache lookup fans out to 3+ servers; with warm caches the resolver answers from memory in under a millisecond. TTLs are the control knob: long TTLs (hours) mean resilience and speed but slow migrations; short TTLs (60 s) mean agility but more load and more exposure to resolver outages. Negative answers (NXDOMAIN) are cached too.

**The root is 13 addresses, not 13 servers.** For protocol-historical reasons (fitting a UDP packet) there are exactly 13 named root identities, `a.root-servers.net` through `m`, run by 12 independent operators (Verisign runs both A and J). Each address is **anycast**: announced via BGP from many places at once, so "asking F-root" really means asking whichever of F's 366 instances is topologically nearest. As of 2026-08-20, root-servers.org counts **2,004 operational instances** — E-root alone has 328, F-root 366, D-root 231. Blowing up one building doesn't dent the root.

**DNS over HTTPS/TLS.** Classic DNS is plaintext UDP on port 53 — your ISP and anyone on-path sees every domain you visit. DoT (RFC 7858) and DoH (RFC 8484, 2018) wrap queries in TLS/HTTPS; DoH makes DNS traffic indistinguishable from web traffic. Cloudflare's 1.1.1.1 (launched April 1, 2018) shipped both, promised query IPs never hit disk with logs wiped in 24 h (KPMG-audited), and measured ~14 ms global average resolution — resolver choice became a privacy and performance decision users can actually make.

Component list (plain text):
- Stub resolver (in your OS) + browser DNS cache
- Recursive resolver (ISP / 1.1.1.1 / 8.8.8.8), with cache
- 13 root server identities (a-m.root-servers.net), ~2,004 anycast instances, 12 operators
- ~1,500 TLD registries' authoritative servers (.com run by Verisign, etc.)
- Domain-owner authoritative nameservers (Cloudflare, Route 53, self-hosted...)
- Record types: A/AAAA (addresses), NS (delegation), CNAME (alias), MX (mail), TXT, SOA
- Transport: UDP/53 classic, DoT 853, DoH 443

## Signature ideas

- **Recursive vs authoritative is the load-bearing distinction.** One kind of server chases answers on your behalf and remembers them; the other kind *is* the answer for its own zone. Nearly every DNS product, outage, and attack makes sense once you know which side it lives on (Dyn 2016 = authoritative side down; your ISP resolver dying = recursive side down).
- **The hierarchy means nobody knows everything.** Root servers don't know where example.com is — only where .com is. Each level hands out referrals, not answers. That's what lets a namespace with hundreds of millions of domains work with no central database and no coordination beyond delegation records.
- **Caching is the real DNS.** The full root→TLD→authoritative walk is the rare path; the overwhelming majority of queries die in a cache within a few milliseconds. The TTL on every record is a distributed-systems contract: "you may believe this for N seconds" — the internet's most successful eventual-consistency deployment.
- **Anycast turns 13 into 2,000.** One IP address announced from hundreds of places means capacity, latency, and DDoS resistance all scale by adding instances, invisibly to clients. The 13-address limit became irrelevant the day the roots went anycast.
- **DNS is a dependency amplifier.** It sits before every other protocol, so a DNS failure masquerades as total failure of whatever sits behind it — and failing clients *retry aggressively*, so DNS outages generate their own thundering herd (1.1.1.1 saw 30x query volume during Facebook's 2021 outage).
- **Encrypting DNS moved trust, not removed it.** DoH hides queries from your ISP but concentrates them at whichever resolver you chose — the debate around it is really a debate about who deserves the metadata.

## Key numbers
- 2,004 root server instances across 13 addresses / 12 operators, as of 2026-08-20 (root-servers.org, 2026; https://root-servers.org/)
- Per-letter instance counts: F=366, E=328, D=231, J=147, I=91 (root-servers.org, 2026; https://root-servers.org/)
- Typical uncached lookup 20-120 ms; cached <1 ms (Cloudflare Learning Center, 2024; https://www.cloudflare.com/learning/dns/what-is-dns/)
- 1.1.1.1 global average latency ~14 ms at launch, fastest public resolver per DNSPerf (Cloudflare, 2018; https://blog.cloudflare.com/announcing-1111/)
- 1.1.1.1 privacy: no query IPs to disk, logs wiped in 24 h, KPMG audit (Cloudflare, 2018; https://blog.cloudflare.com/announcing-1111/)
- DoH standardized as RFC 8484 (IETF, Oct 2018; https://datatracker.ietf.org/doc/html/rfc8484)
- Dyn attack: traffic from "10s of millions of IP addresses," reported peak ~1.2 Tbps, ~18 hours of attack waves, Oct 21 2016 (Dyn statement via ThousandEyes/Wikipedia, 2016; https://www.thousandeyes.com/blog/dyn-dns-ddos-attack, https://en.wikipedia.org/wiki/DDoS_attacks_on_Dyn)
- 30x normal query volume hit 1.1.1.1 during the Facebook outage, most answers still <10 ms (Cloudflare, 2021; https://blog.cloudflare.com/october-2021-facebook-outage/)

## Famous incidents
- **Dyn DDoS — Oct 21, 2016.** The Mirai botnet (IoT cameras/DVRs conscripted via ~60 default passwords) fired attack waves reported around 1.2 Tbps at Dyn, a major *authoritative* DNS provider. Twitter, Spotify, GitHub, Netflix, PayPal and more "went down" for hours across the US East Coast and Europe — their servers were fine; nobody could resolve their names. Teaches: outsourcing authoritative DNS to one provider creates a shared fate, and cheap insecure devices become weapons. (Aftermath: many majors added a second DNS provider.)
- **Facebook — Oct 4, 2021.** Facebook's DNS servers were healthy but their BGP routes were withdrawn (see bgp.md), so every resolver on Earth got SERVFAIL for facebook.com for ~6 hours. Clients and apps retried in a loop, driving 30x load onto 1.1.1.1. Teaches: DNS failure is indistinguishable from total failure, and DNS outages create retry storms that batter *other* infrastructure.
- **The root servers' stress tests.** The 13 root identities have absorbed repeated DDoS attempts (notably 2002 and Nov 30 2015, when several root letters each absorbed on the order of 5M queries/sec per the operators' joint incident report); anycast dispersion meant users barely noticed. Teaches: anycast is the reason "take down the internet's core" attacks stopped being plausible. (RIPE/ICANN incident reports; root-servers.org.)
- **DoH turf war — 2018-2020.** Mozilla's move to default DoH via Cloudflare drew fire from ISPs and governments (UK ISP association infamously nominated Mozilla "Internet Villain") because it blanked their DNS-based monitoring and filtering. Teaches: metadata is power; changing who sees DNS queries is a political act, not just a technical one.

## Visualization hooks
- **The resolution treasure hunt**: an animated query dot leaving a laptop, bouncing resolver → root → TLD → authoritative with speech-bubble referrals ("not me — try these guys"), then racing home with the IP and stamping caches on the way back.
- **Cache-hit sankey**: 1,000 queries enter; ~90%+ die at browser/OS/resolver caches, a trickle reaches the root — makes "caching is the real DNS" visceral.
- **The 13-addresses illusion**: 13 labeled pins that each explode into hundreds of dots across a world map (2,004 total), with anycast arrows pulling each user to their nearest dot.
- **TTL as an hourglass** on each cached record; watch a migration play out as old TTLs drain data-center-A answers and refill with data-center-B.
- **Recursive vs authoritative split-screen**: librarian character running between shelves (recursive) vs the shelves themselves (authoritative); replay Dyn 2016 by knocking over shelves, replay an ISP resolver outage by removing the librarian.
- **Plaintext vs DoH x-ray**: same query drawn twice — once as a readable postcard passing snooping eyes (ISP, coffee-shop WiFi), once as a sealed HTTPS envelope indistinguishable from web traffic.
- **Retry-storm feedback loop**: Facebook outage as a graph — SERVFAILs spawn retries, query volume climbing to 30x on resolvers that had nothing to do with the outage.
- **The delegation pyramid**: root at top ("knows only where TLDs are"), ~1,500 TLDs, hundreds of millions of domains at the base — each layer only knows one hop down.

## Sources
- Root server system live counts — root-servers.org (all 12 operators), 2026. Instance totals, per-letter data, map. Primary. https://root-servers.org/
- "What is DNS?" — Cloudflare Learning Center, 2024. Server types, 8-step lookup, 20-120 ms figure. Primary vendor explainer (page blocks some fetchers; corroborated by Akamai's "What is recursive DNS?" https://www.akamai.com/glossary/what-is-recursive-dns). https://www.cloudflare.com/learning/dns/what-is-dns/
- "Announcing 1.1.1.1" — Cloudflare blog, 2018. Resolver launch, DoH/DoT, privacy commitments, 14 ms figure. Primary. https://blog.cloudflare.com/announcing-1111/
- "October 2021 Facebook outage" — Cloudflare blog, 2021. DNS-side view, SERVFAILs, 30x retry storm, timeline. Primary observer. https://blog.cloudflare.com/october-2021-facebook-outage/
- "The DDoS Attack on Dyn's DNS Infrastructure" — ThousandEyes blog, 2016. Measured view of the Dyn outage. Secondary (independent measurement). https://www.thousandeyes.com/blog/dyn-dns-ddos-attack
- "DDoS attacks on Dyn" — Wikipedia, updated. Aggregates Dyn's statements (10s of millions of IPs, Mirai) and affected-site lists. Tertiary; follow citations. https://en.wikipedia.org/wiki/DDoS_attacks_on_Dyn
- RFC 1034/1035 — IETF, 1987. DNS concepts and implementation; readable and historically great. Primary/RFC. https://datatracker.ietf.org/doc/html/rfc1034
- RFC 8484 — IETF, 2018. DNS over HTTPS. Primary/RFC. https://datatracker.ietf.org/doc/html/rfc8484
