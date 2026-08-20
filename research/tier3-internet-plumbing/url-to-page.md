# What Happens When You Type a URL and Press Enter

## One-line hook
Between your keystroke and the pixels on screen, your request crosses a keyboard bus, an OS, a global name database, three handshakes, dozens of routers, a server farm, and a GPU — usually in under a second, and every stage is a system someone had to invent.

## The core problem
A human types a name ("google.com"); the network only moves numbered packets between machine interfaces. Every layer of the stack exists to bridge that gap: names to addresses (DNS), unreliable packets to reliable streams (TCP/QUIC), plaintext to authenticated encryption (TLS), bytes to structured documents (HTTP), and documents to rendered pixels (the browser engine). The end-to-end story is the single best tour of internet architecture because it touches everything exactly once.

## How it works

End-to-end flow with rough latency budget (assume a warm laptop, ~30-50 ms RTT to the server, nothing cached):

1. **Keyboard → OS (sub-millisecond to ~10 ms).** The key press closes a circuit; a USB keyboard's endpoint is polled by the host controller roughly every 10 ms, and the keycode travels through the HID driver to the focused application. (Source: alex/what-happens-when.)
2. **URL parsing + HSTS check (~0 ms).** The browser decides whether the input is a URL or a search query, extracts scheme/host/path, punycodes non-ASCII hostnames, and checks the preloaded HSTS list — if the domain is on it, the browser upgrades to HTTPS before any network I/O.
3. **DNS resolution (~1 ms cached; ~20-120 ms uncached).** Browser cache → OS cache → hosts file → the configured recursive resolver, which walks root → TLD → authoritative if it has nothing cached. (Details in dns.md.)
4. **ARP / local delivery (~0-1 ms).** Before any packet leaves, the OS needs the MAC address of the next hop (usually the home router) — from the ARP cache or a broadcast ARP request.
5. **TCP three-way handshake (1 RTT, ~30-50 ms).** SYN → SYN-ACK → ACK. No application data moves until it completes. The client's ACK can carry the first TLS bytes, so the cost is one full round trip.
6. **TLS 1.3 handshake (1 RTT, ~30-50 ms).** The client sends its Diffie-Hellman key shares speculatively in the ClientHello instead of waiting for cipher negotiation, so the server can answer with its certificate and finish in a single round trip — down from two in TLS 1.2. Resumed connections can send 0-RTT early data with the first flight, at the cost of replayability (browsers restrict 0-RTT to idempotent GETs). (Source: Cloudflare RFC 8446 post.)
7. **Or: QUIC / HTTP/3 collapses steps 5-6 (1 RTT total).** QUIC runs over UDP, lives in user space, and fuses the transport handshake with the TLS 1.3 handshake — encryption is not optional. Independent streams remove TCP's head-of-line blocking: one lost packet no longer stalls unrelated resources. (Source: Cloudflare HTTP/3 post.)
8. **HTTP request → server (~1 RTT + server think time, ~30-200 ms).** The browser sends `GET / HTTP/2` (or /3) with Host/authority, cookies, and content negotiation headers. The server (often a CDN edge, then possibly an origin) routes, executes application logic, and streams back a status line, headers, and the HTML body — or `304 Not Modified` if the browser's cached copy is still valid.
9. **Parse: bytes → DOM + CSSOM (tens of ms).** The HTML parser tokenizes the stream in chunks (historically ~8 KB) and builds the DOM incrementally; it does not wait for the full document. CSS is parsed into the CSSOM. A synchronous `<script>` without `defer/async` blocks parsing — the classic render-blocking pitfall.
10. **Subresource fetches (parallel, dominates total time).** Discovered stylesheets, scripts, fonts, and images trigger new fetches — each potentially repeating DNS/TCP/TLS for new origins, which is why real pages take 1-3 s while the HTML itself arrived in 200 ms. The preload scanner races ahead of the parser to start these early.
11. **Render pipeline: Style → Layout → Paint → Composite (per frame, budget ~10 ms).** Computed styles are resolved per element; layout computes geometry (widths top-down, heights bottom-up); paint rasterizes text, colors, borders, shadows into layers; the compositor assembles layers on the GPU. At 60 Hz the browser has 16.66 ms per frame, but after browser overhead "all of your work needs to be completed inside 10 milliseconds." Changing only compositor-friendly properties (transform, opacity) skips layout and paint entirely. (Source: web.dev.)
12. **JavaScript keeps the pipeline alive.** Timers, input handlers, and fetches mutate the DOM and re-trigger style/layout/paint — the page is a living system, not a one-shot render.

Component list (plain text):
- Keyboard controller + USB/Bluetooth HID stack
- OS input subsystem → browser process
- Browser: URL parser, HSTS list, DNS cache, socket pool
- Recursive DNS resolver (ISP or 1.1.1.1/8.8.8.8) + root/TLD/authoritative servers
- Home router (NAT), ISP access network, transit/peering links (BGP-chosen path)
- Server side: CDN edge, load balancer, app servers, databases
- Browser engine: HTML parser, CSS parser, JS engine, layout engine, compositor, GPU process

## Signature ideas

- **The stack is a tower of translations.** Name→IP (DNS), IP→MAC (ARP), packets→streams (TCP/QUIC), streams→secure channel (TLS), channel→semantics (HTTP), markup→pixels (rendering). Each layer only trusts the contract of the layer below — that modularity is why the pieces could evolve independently for 40 years.
- **Latency is round trips, not bandwidth.** A page's speed is mostly counted in RTTs: 1 for TCP, 1 for TLS 1.3 (was 2 in 1.2), then request/response. That's why the last decade of protocol work (TLS 1.3, QUIC, 0-RTT) is a war on round trips — and why CDNs win by moving the endpoint closer so each RTT is 10 ms instead of 100.
- **The handshake merger.** QUIC's key trick is refusing to treat transport setup and crypto setup as separate conversations. Fusing them over UDP gets a secure, multiplexed connection in one round trip and makes encryption structurally mandatory. (Cloudflare HTTP/3 post.)
- **Incremental everything.** The browser never waits for completeness: DOM builds as bytes arrive, the preload scanner fetches ahead, paint happens before all resources land. The web feels fast because every stage streams into the next.
- **The pixel pipeline has fast paths.** JS/Style→Layout→Paint→Composite is the full pipe, but paint-only changes skip layout, and transform/opacity changes skip both — the difference between a 60 fps animation and jank. (web.dev.)
- **0-RTT is a deal with the devil.** Sending data before the handshake completes means an attacker can replay it; the mitigation is convention (only safe GETs) plus ticket age checks — a lovely example of a security/latency trade made explicit in protocol design. (Cloudflare RFC 8446 post.)

## Key numbers
- USB keyboard endpoint polled ~every 10 ms (alex/what-happens-when, ongoing; https://github.com/alex/what-happens-when)
- Uncached DNS lookup typically 20-120 ms; cached ~<1 ms (Cloudflare Learning Center figure, widely cited; https://www.cloudflare.com/learning/dns/what-is-dns/, 2024)
- TCP handshake: 1 RTT before data (RFC 793 behavior; https://github.com/alex/what-happens-when)
- TLS 1.3: 1-RTT handshake, 0-RTT resumption; RFC 8446 published 2018-08-10 (Cloudflare, 2018; https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
- TLS 1.2 needed 2 RTTs — TLS 1.3 halves handshake latency (Cloudflare, 2018; same URL)
- Frame budget: 16.66 ms at 60 Hz, ~10 ms usable after browser overhead; 200 ms acceptable for discrete interactions (INP) (web.dev, 2023; https://web.dev/articles/rendering-performance)
- HTML parsed in chunks (~8 KB) as it streams (alex/what-happens-when; https://github.com/alex/what-happens-when)
- HTTP/3 on Cloudflare's edge announced Sept 2019 (Cloudflare, 2019; https://blog.cloudflare.com/http3-the-past-present-and-future/)

## Famous incidents
- **The question itself became famous.** "What happens when you type google.com and press enter" is the archetypal systems interview question; the community answer (alex/what-happens-when, ~40k+ GitHub stars) runs from keyboard circuitry to GPU compositing and shows how deep the "simple" question goes. Teaches: every abstraction you skip is a place you can be surprised.
- **Facebook, Oct 4 2021 — the day step 3 disappeared.** A BGP withdrawal made Facebook's DNS servers unreachable; for the world's browsers the flow died at DNS with SERVFAIL, and retry storms drove 30x load onto public resolvers. Teaches: the early stages of the pipeline are shared infrastructure — when they fail, it looks like *everything* failed. (Cloudflare postmortem; details in bgp.md.)
- **Render-blocking history.** Before `async`/`defer` and the preload scanner matured, a single slow synchronous `<script>` in `<head>` froze parsing for entire pages — the reason "put scripts at the bottom" became folk wisdom, later codified into the critical-rendering-path guidance (web.dev). Teaches: the pipeline is only as fast as its most blocking stage.

## Visualization hooks
- **The full journey as a horizontal timeline** with real millisecond widths: 10 ms input, 30 ms DNS, 40 ms TCP, 40 ms TLS, 60 ms request+response, 100 ms parse/render — instantly shows where time actually goes.
- **Round-trip ping-pong diagram**: client and server as two vertical lines, arrows for SYN/SYN-ACK/ACK, ClientHello/ServerHello, GET/200 — then a second panel where QUIC collapses arrows into one flight.
- **TLS 1.2 vs 1.3 vs QUIC side-by-side**: three columns, count the arrows (2 RTT → 1 RTT → 1 RTT combined → 0-RTT resumed).
- **The parser as a factory conveyor**: bytes in, tokens, DOM nodes assembling into a growing tree, with the preload scanner as a scout running ahead grabbing resources.
- **Pixel pipeline as five gates** (JS → Style → Layout → Paint → Composite) with three animated balls taking the full path, the skip-layout path, and the composite-only express lane.
- **Layer/altitude map**: the same request drawn at 7 altitudes (app, TLS, TCP, IP, link, physical) — one packet shown wearing nested envelopes.
- **A "what can go wrong here" annotation pass**: the same timeline with failure stickers per stage (NXDOMAIN, RST, cert error, 500, jank).
- **Head-of-line blocking**: HTTP/2 as one pipe where a dropped brick stops the whole conveyor vs QUIC as parallel pipes where only one stalls.

## Sources
- "What happens when..." — alex/what-happens-when, GitHub, 2013-ongoing. Community-maintained deep answer, keyboard to GPU. Secondary but exceptionally vetted; the canonical reference for this question. https://github.com/alex/what-happens-when
- "A Detailed Look at RFC 8446 (a.k.a. TLS 1.3)" — Cloudflare blog, 2018. 1-RTT handshake mechanics, 0-RTT and replay risk, removed legacy crypto. Primary (Cloudflare co-developed deployment). https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/
- "HTTP/3: the past, the present, and the future" — Cloudflare blog, 2019. QUIC over UDP, merged handshakes, head-of-line blocking, connection migration. Primary. https://blog.cloudflare.com/http3-the-past-present-and-future/
- "Rendering performance" — web.dev (Google Chrome team), 2023. Pixel pipeline stages, frame budget, skippable stages. Primary (browser vendor). https://web.dev/articles/rendering-performance
- "What is DNS?" — Cloudflare Learning Center, 2024. The 20-120 ms lookup figure and 8-step resolution. Primary vendor explainer (note: page blocks some fetchers; figures corroborated by Akamai and freeCodeCamp explainers). https://www.cloudflare.com/learning/dns/what-is-dns/
- RFC 8446 — IETF, 2018. TLS 1.3 specification. Primary/RFC. https://datatracker.ietf.org/doc/html/rfc8446
- RFC 9000 — IETF, 2021. QUIC transport specification. Primary/RFC. https://datatracker.ietf.org/doc/html/rfc9000
