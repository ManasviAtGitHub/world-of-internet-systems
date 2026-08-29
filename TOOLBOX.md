# The Toolbox — component design

The DSA-revision tier of World of Internet Systems. One structure per piece,
honestly simulated, anchored to the 30 systems where the reader has already met
it under fire. This document is the format contract: every piece is assembled
from these components, in this order, to this standard.

**Naming policy (site-wide):** no cross-page numbering — no T-NN, no EP-NN.
Names are the identity ("The Filter", "the Timeline episode"). Within-page
badges (`LIVE`, `EXP-01`, `REP-01`) are fine: they order experiments inside a
page, which is real structure. Traversal order comes from the sidebar and,
later, the Atlas reading map.

---

## The story spine

Every piece runs the same seven beats, top to bottom:

1. **Header** — eyebrow, title, dek, marginal
2. **The interview card** — where you've ground this before
3. **Hero (LIVE)** — the structure itself, really built, operating
4. **Reading beats + the REP** — predict before you run, then get graded
5. **Production stories (EXP-01, EXP-02)** — the structure under real fire, with episode backlinks
6. **The revision card** — questions for the you that comes back in three months
7. **Sources + footer** — research citations; name-based navigation

The reference implementation of all seven is `episodes/bloom.html` ("The Filter").
Copy its `<style>` and JS scaffold verbatim; the `.tbx` block is the only CSS
this tier adds to the series design system.

---

## Component contracts

### 1. Header
- Eyebrow: `World of Internet Systems · The Toolbox`
- Title: the structure, literally — "The Heap", "The Trie". No brand in the title.
- Dek: names the places in the 30 systems where this structure is running right
  now, states the structure's one asymmetry in a sentence, and closes with the
  tier signature: *"Predict first, then run it. That's the rep."* (vary the
  phrasing, keep the "rep" close).
- Marginal: the honest-simulation disclosure ("everything below computes for
  real") plus the one-line tier explainer.

### 2. The interview card (`.tbx`)
- `k`-line: `The interview card — where you've ground this before`
- Three arrow-marked bullets: real interview framings of this structure (the
  phrasings people actually grind on — LeetCode/system-design idiom).
- Facts grid (a `<dl>`, label column + value column, ruled off from the
  bullets): `Pattern` / `Space` / `Time` / `The catch` — the four facts a
  revision pass wants in ten seconds. The catch is mandatory: every structure
  has one and it's what interviews probe. Never a run-on met-line — the grid
  is the component.

### 3. Hero — the structure, honestly built
- A **real implementation** operating live on canvas — never a diagram of one.
  The filter really hashes; the heap really sifts; the trie really walks edges.
- Must make the structure's **invariant or asymmetry visible** (dark bit =
  certain "no"; parent beats children; levels thin geometrically).
- Metrics row includes one **invariant-truth metric** stated absolutely
  ("FALSE NEGATIVES 0 — IMPOSSIBLE, BY CONSTRUCTION").
- Engineering: `reset()` spawns at least one event so the first paint is never
  blank; static HTML metric defaults are also written by `setM` in `update()`.

### 4. The REP (predict-before-run)
The tier's defining mechanic. Contract:
- Badge `REP-01` (`REP-02` if a piece earns two). Choice buttons in the panel
  head (2–4 options, `data-act`), plus RESET.
- The canvas **refuses to measure until the reader commits** — it shows the
  question and waits. No spectating.
- After the pick: an honest measurement to a stated sample size, then a verdict
  line grading `YOUR CALL` vs `MEASURED` vs `THE FORMULA SAYS`.
- Chart overlay where the topic has a curve: plot the law, drop both markers
  (YOUR CALL, MEASURED) onto it.
- Three prediction shapes, pick what fits: **magnitude** (which order of
  magnitude?), **ranking** (which of these two worlds wins?), **threshold**
  (where does it collapse?).
- The note ends with the **pocket line**: the ONE number or sentence worth
  memorizing (The Filter's: *false positives ≈ 0.62 per bit per key*).
- Measurement honesty: rates measured against a **sliding window**, never a
  lifetime average that mixes in the warm-up phase.

### 5. Production stories (EXP-01, EXP-02)
- One or two experiments anchoring the structure in named systems, with
  **name-based backlinks** (`<a href="../index.html#/…" target="_top">`) into
  the episodes where the reader already met it.
- When comparing policies, run a **paired world**: identical event stream
  through both, side by side, so differences are causal, not sampled.
- Sliders and toggles follow the series formula (naive collapses → one switch
  fixes → sliders are yours).
- Showcase annotations are deterministic (a fixed row, a fixed moment) — never
  time-cycled labels that flicker under sampling.

### 6. The revision card (`.tbx`)
- `k`-line: `The revision card — for the you that comes back in three months`
- Three or four questions, each answerable **from something the reader just
  watched** — not from outside reading; each with a parenthetical `.met` nudge.
- Facts grid (same `<dl>` component as the interview card), one row:
  `Seen under fire` → the episode links, name-based (`Title · System`).

### 7. Sources + footer
- Sources cite the real engineering (papers, engineering blogs) exactly like
  the episodes do.
- Footer, name-based: `World of Internet Systems · NEXT IN THE TOOLBOX →
  <a>The …</a> · <b>tease</b>` once a next piece exists; until then, the
  planned-instruments line + back to the index. All links `target="_top"`.

---

## Engineering contract (unchanged from the series)

Standard scaffold: `mulberry32` seeded RNG, `expRand`, dpr-aware `fitCanvas`,
CSS color map, `setM` metric writer, `register()` with IntersectionObserver +
`data-run`/`data-reset`, rAF loop with dt clamp 0.05, `relayout()` +
ResizeObserver. Set `ctx.textAlign` explicitly before every `fillText` group
(alignment leaks between draw branches). Ship only after: awk-extract →
`node --check` → headless harness **ALL CHECKS PASS** → probe screenshots at
1100px and 700px (default + interacted) → label audit clean.

---

## The coverage plan — eleven pieces

Built in waves; order below is sidebar order. Each row fixes the piece's hero,
REP question, production anchors, and pocket line before build starts.

| Piece | Hero shows | REP asks | Production stories (anchors) | Pocket line |
|---|---|---|---|---|
| **The Filter** ✅ | Bloom filter filling; dark bit = certain no | FP rate at 8 bits/key? | LSM absent-key seeks (The Guild, The Graph); one-hit-wonder admission (The Edge) | FP ≈ 0.62 per bit per key |
| **The Heap** ✅ | Live sift-up/sift-down; parent-beats-children invariant | Comparisons for top-100 of 1M: sort vs heap? | k-way merge of 12 partition lists (The Graph, The Timeline); top-K pools (The Board) | top-k of n is n·log k — and log 100 ≈ 7 |
| **The LSM Tree** ✅ | Memtable → flush → sorted runs → compaction, live | Write amplification after N flushes? | ScyllaDB under raids (The Guild); RocksDB feeds (The Graph); pairs with The Filter | sequential wins 100×; compaction is the invoice |
| **The Hash Table** | Buckets filling; birthday collisions; a live resize storm | Load factor where probe length doubles? | the memcached wall (The Board); O(1) likes (The Swipe) | expected probes ≈ 1/(1−α) |
| **The Trie** | Character-edge walk; longest-prefix match live | Nodes shared by 1,000 words vs flat storage? | longest-prefix routing (The Border runs one already); autocomplete | cost = key length, not table size |
| **The Skip List** | Coin-flipped express lanes thinning geometrically | Expected levels at 1M entries? | Redis sorted sets / leaderboards (The Front Page) | a linked list with log n express lanes |
| **The Counter** (HyperLogLog) | 12KB counting millions of distinct items | Error at 2^14 registers? | unique viewers/visitors (The Front Page, The Upload) | error ≈ 1.04 ⁄ √m |
| **The Sketch** (Count-Min) | Heavy hitters surfacing from a stream | Overestimate bound for a cold key? | hot-key detection (The Guild raid); trending (The Front Page) | never undercounts; over by at most εn |
| **The B-Tree** | The duel: same workload vs the LSM, page I/O counted | Which engine wins at 90% reads? | MySQL under The Calendar, The Drop, The Board | three or four levels hold a billion keys |
| **The Eviction** (LRU & co) | Hashmap + linked list really evicting under Zipf | Hit rate at cache = 10% of catalog? | one-hit wonders (The Edge, The Filter); Open Connect (Press Play); parser cache (The Encyclopedia) | under Zipf, recency is a free prophet |
| **The Queue** (capstone) | ρ/(1−ρ) — the law behind every collapse in the series | Wait time at 95% utilization? | every episode; the universal knee | the knee lives at ~80% utilization |

Atlas-only topics (already taught by full episodes, cross-indexed rather than
rebuilt): consistent hashing, Raft, the inverted index, random walks, Wilson
score, quadtree/geo-cells, the bitfield, token buckets, idempotency keys,
Fisher–Yates, interval booking, topological sort, regex automata, bit packing,
vector clocks, power-of-two-choices.

## Site integration

- Sidebar: group `The toolbox — DSA, under fire`; shipped pieces are links,
  planned pieces are `IDEA` rows in build order.
- Landing page: once three pieces exist, add a Toolbox card section beneath
  the thirty.
- The Atlas (separate future build) cross-indexes structure ↔ episode both ways
  and carries the reading paths.
