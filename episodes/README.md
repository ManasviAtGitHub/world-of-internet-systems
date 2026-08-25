# World of Internet Systems — Episodes

Interactive simulation episodes. Each is one self-contained HTML file (no dependencies,
no network). Research base: `../research/`.

**Live site (GitHub Pages, canonical):** https://manasviatgithub.github.io/world-of-internet-systems/
— served from the `gh-pages` branch. **Deploy = push both branches:**
`git push origin main && git push origin main:gh-pages`

Episodes are now full HTML documents (doctype + html/head/body). NOTE: the Claude
Artifact publisher wraps files in its own skeleton and expects fragments WITHOUT
doctype/html/head/body — if an episode is ever republished as an artifact, strip the
outer shell first. The artifact URLs below predate this change and still work.

## Published

| # | Episode | File | Artifact URL |
|---|---------|------|--------------|
| 01 | The Load Balancer | `load-balancing.html` | https://claude.ai/code/artifact/138fdb2a-d3c6-4ec1-a348-e50380714df4 |
| 02 | The Hash Ring | `consistent-hashing.html` | https://claude.ai/code/artifact/4d043cb1-d1cb-4a8d-9081-6a7a60130bc0 |
| 03 | The Timeline (Twitter fan-out) | `twitter-timeline.html` | https://claude.ai/code/artifact/83872498-adb2-43ae-a05a-c17e3c8eb74c |
| 04 | The Log (Kafka) | `the-log-kafka.html` | Pages only |
| 05 | The Name (DNS) | `dns.html` | Pages only |
| 06 | Press Play (Netflix) | `netflix.html` | Pages only |
| 07 | The Message (WhatsApp) | `whatsapp.html` | Pages only |
| 08 | The Border (BGP) | `bgp.html` | Pages only |
| 09 | The Match (Uber) | `uber.html` | Pages only |
| 10 | The Index (Google Search) | `google-search.html` | Pages only |
| 11 | The Quorum (Amazon Dynamo) | `dynamo.html` | Pages only |
| 12 | The Upload (YouTube) | `youtube.html` | Pages only |
| 13 | The Feed (Instagram) | `instagram.html` | Pages only |
| 14 | The Page (URL → pixels) | `url-to-page.html` | Pages only |
| 15 | The Edge (CDN & anycast) | `cdn-edge.html` | Pages only |

Flagship tier complete. Next planned: **The Ladder — databases at scale** (closes plumbing).

Extractor gotcha (learned on EP14): the awk script-extraction drops any JS line
containing a literal `<script>` token (e.g. inside a canvas-label string) — avoid
that exact substring in strings, or the headless harness sees a truncated file.

Editorial lesson from Ep03 (keep): op-count economics do NOT justify the celebrity
threshold — per follow edge, follower count cancels out of the push-vs-pull comparison.
The threshold is a latency promise (no single fanout job may exceed the delivery SLA).
The episode says so explicitly; never re-introduce a "cost minimum" framing for it.

## Series design system ("the notebook and the instrument")

- Reading layer = engineer's field journal: cool paper `#EEF1F4` / ink `#18222E`
  (dark reading theme exists), serif prose (Iowan Old Style/Palatino stack), faint dot grid.
- Simulation panels = dark instruments, **theme-invariant**: panel `#0C1622`, lines `#24384F`,
  mono labels (ui-monospace). Traffic/particles: signal amber `#F2A93B`.
- Chart series (CVD-validated pair, dark surface): data amber `#C48224` + relay cyan `#3B9EBD`.
  Fault red `#E4604E` reserved for drops/overload only.
- Episodes are numbered experiments (EXP-01…): naive design → visible collapse →
  one-switch fix → viewer sandbox. Metrics use sliding time windows (not all-history).
- No characters (deliberate — see essence-of-agents for that voice). Borrow-worthy from
  that project: the Timeline+scrubber pattern (`assets/anim.js`) for future scripted
  narrative episodes (message journeys, outage replays).

## Verification workflow (do this for every episode)

0. First line of every episode file must be `<meta charset="utf-8">` — the artifact wrapper
   supplies encoding, but local viewing (file:// or a plain server) falls back to
   Windows-1252 and mojibakes every — · × ρ without it.

1. Extract inline JS: `awk '/<script>/{f=1;next}/<\/script>/{f=0}f' episode.html > sim.js`
2. `node --check sim.js`
3. Headless harness (see scratchpad harnesses for the pattern): stub DOM/canvas/rAF,
   drive a controlled clock, assert the physics (remap fractions, p99 dynamics,
   recovery after the fix). Both episodes shipped only after ALL CHECKS PASS.
4. Serve locally + probe painted pixels via `getImageData` (artifact iframes may lay out
   late; every episode needs the ResizeObserver relayout self-heal — see the bottom of
   either episode's script).
