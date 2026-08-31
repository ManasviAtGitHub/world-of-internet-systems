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
| 06 | [Press Play — Netflix](episodes/netflix.html) | Remove Open Connect and the backbone drowns; the 3 AM fill beats the premiere miss-storm; be the Chaos Monkey |
| 07 | [The Message — WhatsApp](episodes/whatsapp.html) | Store-and-forward vs flaky phones; the C10K wall vs 2M connections on one box; Sender Keys' ×256 uplink saving |
| 08 | [The Border — BGP](episodes/bgp.html) | Real path-vector routing: the two-stage cable cut, the 2008 hijack duel, RPKI, and Facebook's self-erasure with the ×30 retry storm |
| 09 | [The Match — Uber](episodes/uber.html) | Hex index vs scanning the city (114× capacity); the 2-second batch that shortens pickups 8%; surge as a thermostat |
| 10 | [The Index — Google Search](episodes/google-search.html) | A real inverted index (×1,000, byte-identical); a real random surfer vs the keyword stuffer; the tail at scale, hedged |
| 11 | [The Quorum — Amazon's Dynamo](episodes/dynamo.html) | The R+W>N dial with staleness measured to exactly zero; sloppy quorum + hinted handoff; vector clocks vs last-write-wins |
| 12 | [The Upload — YouTube](episodes/youtube.html) | Chunked-parallel transcoding (live in 5 min, not 60); the thumbnail meltdown vs packed storage; adaptive bitrate vs the dips |
| 13 | [The Feed — Instagram](episodes/instagram.html) | The 64-bit ID minted & decoded live; kill the ID service nobody needed; sort/route verified; cursor vs OFFSET pagination |
| 14 | [The Page — URL to pixels](episodes/url-to-page.html) | The scrubber walk; the war on round trips (TLS 1.2 → QUIC 0-RTT); H2 head-of-line vs H3; the blocking script |
| 15 | [The Edge — CDN & anycast](episodes/cdn-edge.html) | Anycast vs unicast; the 78× cache funnel; the 31.4 Tbps storm at 6% headroom; the 2019 regex bomb with real engines |
| 16 | [The Ladder — databases at scale](episodes/databases.html) | The scaling climb; the vanishing comment; the async write that evaporated; a real Raft election that never splits the brain |
| 17 | [The Guild — Discord](episodes/discord.html) | A community as one Elixir process hits its quadratic wall (Manifold un-jams it); request coalescing decouples raid load from the database; the Go GC quake vs Rust's flat line |
| 18 | [The Workspace — Slack](episodes/slack.html) | One message worldwide in ~500ms, persist-first; the rtm.start truck vs Flannel's diet; the reconnect storm absorbed at the edge; the wedged Redis queue Kafka un-wedged |
| 19 | [The Playlist — Spotify](episodes/spotify.html) | The P2P era's 265ms illusion (one byte in eleven paid for); the event river with bulkheads; a Discover Weekly genuinely computed — kill the audio model and new releases vanish |
| 20 | [The Meeting — Zoom](episodes/zoom.html) | The server that refuses to watch: MCU melts at ~97, the router holds 15× more; simulcast detaches downlink from meeting size; the 30× cliff of March 2020, provisioning dial in hand |
| 21 | [The Folder — Dropbox](episodes/dropbox.html) | A file is a list of hashes: whole-file sync drowns where blocks nap; the teammate's copy costs 0 bytes; LAN sync makes the office the CDN; replicate-fresh vs erasure-code-cold race a repair |
| 22 | [The Calendar — Airbnb](episodes/airbnb.html) | Two guests race one calendar (atomic commit vs the checkout window; iCal stays racy); the Monorail split by 1% comparison; the two-sided ranker that asks if the host will say yes |
| 23 | [The Charge — Stripe](episodes/stripe.html) | Exactly once over an at-least-once wire: the idempotency key vs blind retries; the four-layer bouncer that never sheds a charge; the version time machine serving 2011 fresh from 2026 |
| 24 | [The Drop — Shopify](episodes/shopify.html) | Same hardware, cut two ways: a shared DB breaks every shop, pods contain the fire; the Storefront Renderer's 45ms p75 emerges from its caches; the microservices tax, compounded honestly |
| 25 | [The Onsale — Ticketmaster](episodes/ticketmaster.html) | 14M people, 2M seats: the TTL hold that hurts on both ends; the waiting-room lottery that beats the bots; a cache wall protecting a 1970s VAX core |
| 26 | [The Front Page — Reddit](episodes/reddit.html) | The real hot formula live; precompute-everything vs sort-on-read; Wilson confidence vs the old giants; r/place as one 4-bit bitfield |
| 27 | [The Board — Pinterest](episodes/pinterest.html) | The 64-bit ID that routes itself; the feed as a scored pool, not a timeline; real Pixie random walks with the restart dial; the join that can't shard vs 100 key lookups |
| 28 | [The Graph — LinkedIn](episodes/linkedin.html) | Fan-out on read done seriously: the slowest of 12 partitions sets the pace; push melts when every employer is a celebrity; scoring at the leaves; the atomic swap with zero mixed reads |
| 29 | [The Swipe — Tinder](episodes/tinder.html) | The world cut to equal people (a measured U-curve picks the cell size); the simultaneous-swipe race cured by a pair-keyed stream; polling priced against a 5-byte nudge |
| 30 | [The Encyclopedia — Wikipedia](episodes/wikipedia.html) | The finale: the read:write funnel; purge-don't-expire (the vandalism incident, both worlds); replicate the hits / partition the tail; a 2M-page blast radius paid lazily — and the series' closing argument |

**THE SERIES IS COMPLETE** — 30 systems, honestly simulated: every collapse collapsed by arithmetic, every fix fixed the arithmetic.

## Structure

- `index.html` — the home shell: collapsible systems index, episodes load in place
- `episodes/` — one self-contained HTML file per episode (no dependencies, no network);
  see `episodes/README.md` for the design system and the mandatory verification workflow
- `research/` — the sourced research base behind the series: 29 systems across three
  tiers, every scale number carrying its year and source URL

## The toolbox — DSA, under fire

A revision tier connecting classic data structures to the 30 systems above. Each
piece: interview card, a live honestly-implemented structure, a predict-before-run
experiment (measurement gated on committing to a call), production stories with
episode backlinks, and a revision card. Format contract: `TOOLBOX.md`.

| Piece | Structure | The rep |
|---|---|---|
| The Filter | Bloom filters | call the false-positive rate at 8 bits/key |
| The Heap | binary heap | call the comparison bill for top-100 of 1M |
| The LSM Tree | log-structured merge tree | call the write amplification, fanout 10, 3 levels |
| The Hash Table | open addressing + chaining | call the load factor where probes hit 10x |
| The Trie | prefix tree / radix | call how many nodes English actually needs |
| The Skip List | randomized express lanes | call one search's bill in a million entries |
| The Counter | HyperLogLog | call the error of 12 KB counting a million distinct |

Planned: Count-Min, the B-tree, LRU eviction, and the queue as capstone.

## The rules of the series

1. **Simulations are honest** — queueing behavior emerges from real arithmetic
   (Poisson arrivals, service rates, backlog), never from a script.
2. **Every number is sourced** — engineering blogs, papers, conference talks, cited
   per episode.
3. **Every episode is verified before it ships** — the exact page code is run headlessly
   against behavioral assertions (the collapse must collapse, the fix must fix).
