# World of Internet Systems

**Live site:** https://manasviatgithub.github.io/world-of-internet-systems/

A visualization series about the system designs behind the internet's most popular
services — that doesn't just diagram them, it **runs** them. Each episode is a sequence
of live, honest simulations: the naive design visibly collapses, one switch applies the
real fix, and the sliders are yours.

## Episodes

| # | Episode | The proof |
|---|---------|-----------|
| 01 | [The Load Balancer](episodes/load-balancing.html) | One server hits the 1/(1−ρ) wall; round robin betrayed by a slow server; the stale-data stampede; power of two choices |
| 02 | [The Hash Ring](episodes/consistent-hashing.html) | mod-N scrambles 83% of keys on one failure, the ring moves 17% (the provable minimum); rolling-restart endurance; virtual nodes |
| 03 | [The Timeline — Twitter](episodes/twitter-timeline.html) | 60:1 read/write asymmetry decides push over pull; the celebrity fan-out clog; the hybrid fix; the 5-second rule |
| 04 | [The Log — Kafka](episodes/the-log-kafka.html) | The same spike: direct coupling loses messages, the log converts failure into lag; crash + rewind; N×M → N+M |
| 05 | [The Name — DNS](episodes/dns.html) | The resolution walk; caching off = the root melts; TTL cuts both ways; the Dyn 2016 attack replayed in TTL order |

Next: **Press Play — Netflix and the CDN inside your ISP** (sidebar shows the roadmap).

## Structure

- `index.html` — the home shell: collapsible systems index, episodes load in place
- `episodes/` — one self-contained HTML file per episode (no dependencies, no network);
  see `episodes/README.md` for the design system and the mandatory verification workflow
- `research/` — the sourced research base behind the series: 29 systems across three
  tiers, every scale number carrying its year and source URL

## The rules of the series

1. **Simulations are honest** — queueing behavior emerges from real arithmetic
   (Poisson arrivals, service rates, backlog), never from a script.
2. **Every number is sourced** — engineering blogs, papers, conference talks, cited
   per episode.
3. **Every episode is verified before it ships** — the exact page code is run headlessly
   against behavioral assertions (the collapse must collapse, the fix must fix).
