# Twitter / X

## One-line hook
Twitter is the canonical lesson in the push-vs-pull tradeoff: it is cheaper to do millions of writes when a tweet is posted than to do one expensive read when a timeline is opened — until a celebrity shows up and breaks the math.

## The core problem
Deliver a personalized, near-real-time timeline to hundreds of millions of users, where the write side is tiny (thousands of tweets/second) but the read side is enormous (hundreds of thousands of timeline requests/second), and where the follow graph is wildly asymmetric — most users have a few hundred followers, a few have tens of millions. A naive "SELECT tweets FROM everyone I follow ORDER BY time" query cannot survive at 300K+ reads/sec, so the timeline must be precomputed — but precomputing for a 30M-follower account means tens of millions of writes for one tweet. Twitter's whole architecture is the negotiation between those two failure modes.

## Architecture overview
End-to-end data flow (classic ~2012-2013 architecture, per Raffi Krikorian's "Timelines at Scale" talk and the HighScalability writeup):

1. **Write path**: User hits Tweet → request lands on the Tweet write API → tweet is assigned a unique 64-bit **Snowflake ID** (time-ordered, generated without coordination) → tweet text/metadata is persisted to the tweet store (MySQL, later the **Manhattan** distributed database) → the tweet ID is handed to the **fanout service**.
2. **Fanout-on-write**: The fanout daemon queries the social graph service (Flock) for the author's follower list, then iterates through followers in batches (~4,000 destinations at a time), inserting the tweet ID (not the text — just ID + author ID + ~4 bytes of flags) into each follower's **home timeline cache**: a Redis list capped at ~800 entries, replicated 3x across machines. Only active users have materialized timelines.
3. **Read path**: User opens the app → timeline service reads their Redis timeline list (a handful of IDs), then hydrates the tweets and users in parallel from cache/storage → target was tweet visible to followers in under ~5 seconds.
4. **Search path (the mirror image)**: The same tweet goes to **Earlybird**, a modified Lucene index kept in RAM. Search is write-cheap (index on one partition, O(1) write) and read-expensive (scatter-gather across all index partitions), exactly the opposite tradeoff from the timeline.
5. **Ranking era (2016→, fully documented in the 2023 open-sourced "the-algorithm")**: The For You timeline is built by **Home Mixer** (Scala, on the Product Mixer framework): candidate sourcing pulls ~1,500 tweets (roughly half in-network via RealGraph/Earlybird, half out-of-network via GraphJet/UTEG random walks, SimClusters community embeddings, TwHIN graph embeddings) → a logistic-regression **light ranker** prunes → a ~48M-parameter neural **heavy ranker** (MaskNet) scores engagement probabilities → heuristics (author diversity, visibility filtering, blue-verified boosts) → blend with ads and recommendations.

Component list (plain text):
- Tweet API / write service
- Snowflake ID generation service
- Social graph service (Flock/FlockDB)
- Fanout daemons
- Home timeline Redis cluster (lists of tweet IDs, ~800 cap)
- Tweet store: MySQL sharded → Manhattan (2014)
- Timeline service (read path, hydration)
- Earlybird search index + Blender (search scatter-gather)
- Firehose / streaming
- Finagle RPC (JVM services layer)
- Home Mixer + candidate sources (RealGraph, GraphJet, SimClusters, TwHIN) + light/heavy rankers

## Signature ideas
- **Fan-out-on-write (push) timelines.** Precompute every user's home timeline at tweet-write time by inserting IDs into followers' Redis lists. Reads become O(1) list fetches at ~1ms p50. The cost: one tweet can trigger millions of writes, and delivery is eventually consistent (replies could appear before the original during fanout lag).
- **The celebrity problem and the push/pull hybrid.** For high-follower accounts, fanout explodes (Lady Gaga at ~31M followers meant tens of millions of Redis inserts per tweet). Twitter's fix: don't fan out celebrities; instead merge their tweets into followers' timelines at read time — a hybrid where normal users are pushed and celebrities are pulled. This is now the textbook example of hybrid feed delivery.
- **Snowflake IDs.** 64-bit, roughly time-sortable IDs generated with zero coordination: ~41 bits of millisecond timestamp + 10 bits of worker ID (assigned via ZooKeeper) + 12 bits of per-worker sequence, allowing ~4,096 IDs/ms/worker. Invented in 2010 when moving tweets off a single MySQL auto-increment; since copied by Instagram, Discord, and half the industry.
- **The fail whale and the Ruby→JVM rewrite.** The 2010 World Cup melted the Ruby-on-Rails monolith ("Monorail"); Twitter spent 2010-2013 decomposing it into JVM (Scala/Java) services communicating via Finagle RPC. The payoff moment: 143,199 tweets/sec during the 2013 "Castle in the Sky" airing in Japan — ~25x normal load — with no visible blip.
- **Manhattan: multi-tenant distributed database.** Built in-house (announced 2014) after Cassandra pain, serving many internal teams from one system: pluggable storage engines (memory-mapped seadb for read-only bulk data, SSTable/B-tree for read-write), eventual consistency by default with strong-consistency options, ZooKeeper only for topology, not the data path.
- **The open-sourced ranking pipeline (2023).** Rare full disclosure of a production recommender: ~1,500 candidates per request from a pool of hundreds of millions, ~50/50 in-network/out-of-network, GraphJet real-time bipartite graph random walks (~15% of For You tweets), SimClusters' ~145K communities, then light ranker → MaskNet heavy ranker whose final score is a weighted sum of ~10 engagement probabilities (e.g., a predicted report weighted −369 vs +0.5 for a like).

## Key numbers
- 150M+ active users, 300K+ QPS timeline reads (1ms p50, 4ms p99), ~400M tweets/day (~5K/sec avg, peaks 7-12K/sec) — 2013, Raffi Krikorian "Timelines at Scale" via HighScalability: https://highscalability.com/the-architecture-twitter-uses-to-deal-with-150m-active-users/
- Home timeline = Redis list of ~800 entries; fanout processes ~4K destinations per batch; delivery target under ~5 seconds — 2013, same HighScalability writeup: https://highscalability.com/the-architecture-twitter-uses-to-deal-with-150m-active-users/
- 143,199 tweets/sec peak (Aug 3, 2013, "Castle in the Sky" TV airing in Japan), ~25x the ~5,700/sec steady state — 2013, Twitter engineering blog: https://blog.twitter.com/engineering/en_us/a/2013/new-tweets-per-second-record-and-how
- Snowflake: 64-bit IDs = timestamp + worker + sequence; ~4,096 IDs/ms/machine; announced June 2010 — Twitter engineering blog: https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake
- Twitter Redis fleet: 105TB RAM, 39M QPS, 10,000+ instances — 2014, HighScalability: https://highscalability.com/how-twitter-uses-redis-to-scale-105tb-ram-39mm-qps-10000-ins/
- Manhattan handling ~6,000 tweets/sec of writes plus derived data; announced April 2014 — Twitter engineering blog: https://blog.twitter.com/engineering/en_us/a/2014/manhattan-our-real-time-multi-tenant-distributed-database-for-twitter-scale
- For You pipeline: ~1,500 candidates/request, ~50% in-network / 50% out-of-network, heavy ranker ~48M parameters, SimClusters ~145K communities, GraphJet ~15% of For You tweets — 2023, Twitter engineering blog + the-algorithm repo: https://blog.twitter.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm and https://github.com/twitter/the-algorithm

## Evolution timeline
- **2006-2009**: Ruby on Rails monolith + MySQL; rapid growth, frequent outages — the "fail whale" era (2010 World Cup as the breaking point).
- **2010**: Snowflake IDs (escape from single MySQL auto-increment); beginning of JVM services; Earlybird real-time search.
- **2010-2013**: Great re-architecture — Rails monolith decomposed into Scala/Java SOA on Finagle RPC; Redis-based fanout timelines mature; 2013 record traffic handled cleanly.
- **2012-2013**: Hybrid push/pull for high-follower accounts emerges to tame celebrity fanout.
- **2014**: Manhattan replaces Cassandra/MySQL for many storage use cases; multi-tenant in-house database.
- **2016**: Algorithmic (ranked) home timeline replaces strict reverse-chronological by default.
- **2017**: "The Infrastructure Behind Twitter: Scale" post — hybrid Mesos/Aurora private cloud, hundreds of thousands of cores.
- **2023**: "the-algorithm" open-sourced — candidate sourcing → light ranker → heavy ranker → Home Mixer pipeline made public.

## Visualization hooks
- Split-screen "push vs pull": one tweet exploding into millions of Redis inserts (push) vs one timeline read fanning into a scatter-gather across every index shard (pull); search and timeline as mirror images.
- The celebrity problem: a normal user's tweet as a small firework vs Lady Gaga's as a 31M-arrow burst; then the hybrid fix — celebrity tweets held back and merged at read time.
- Anatomy of a Snowflake ID: a 64-bit ruler segmented into timestamp / worker / sequence, with "sortable by time" annotated.
- The 800-slot conveyor belt: each user's home timeline as a fixed-length Redis list, old tweet IDs falling off the end.
- Race condition frame: a reply arriving in a follower's timeline before the celebrity's original tweet.
- The 2013 spike chart: flat ~5.7K TPS line, then the 143,199 TPS needle during the Castle in the Sky airing — annotated "no fail whale."
- The 2023 algorithm as a funnel: hundreds of millions of tweets → 1,500 candidates (50/50 in/out network) → light ranker → 48M-parameter heavy ranker → ~dozens on screen, with the −369 report weight called out.
- Fail whale → JVM: timeline of the Rails Monorail being carved into services (a whale being lifted by birds, dissolving into microservice boxes).

## Sources
- "Announcing Snowflake" — Twitter Engineering Blog, 2010. The original ID-generation post: uncoordinated 64-bit time-ordered IDs, ZooKeeper-assigned worker IDs. Primary. https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake
- "New Tweets per second record, and how!" — Twitter Engineering Blog, 2013. The 143,199 TPS record and the Ruby→JVM/SOA re-architecture narrative. Primary. https://blog.twitter.com/engineering/en_us/a/2013/new-tweets-per-second-record-and-how
- "Manhattan, our real-time, multi-tenant distributed database for Twitter scale" — Twitter Engineering Blog, 2014. Design goals, storage engines, multi-tenancy. Primary. https://blog.twitter.com/engineering/en_us/a/2014/manhattan-our-real-time-multi-tenant-distributed-database-for-twitter-scale
- "The Architecture Twitter Uses to Deal with 150M Active Users, 300K QPS..." — HighScalability, 2013. Detailed writeup of Raffi Krikorian's "Timelines at Scale" InfoQ talk: fanout mechanics, Redis timeline structure, celebrity problem, hybrid plans. Secondary (but faithful to a primary talk). https://highscalability.com/the-architecture-twitter-uses-to-deal-with-150m-active-users/
- "Timelines at Scale" — Raffi Krikorian (VP Platform Engineering), InfoQ/QCon talk, 2012-2013 (slides on SlideShare/Speaker Deck). The primary source behind the above. Primary. https://www.slideshare.net/slideshow/raffi-krikorian-twitter-timelines-at-scale/24040648
- "How Twitter Uses Redis to Scale" — HighScalability, 2014. Redis fleet scale numbers and timeline cache details. Secondary. https://highscalability.com/how-twitter-uses-redis-to-scale-105tb-ram-39mm-qps-10000-ins/
- "Twitter's Recommendation Algorithm" — Twitter Engineering Blog, March 2023. The official explanation of the For You pipeline accompanying the open-source release. Primary. https://blog.twitter.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm
- twitter/the-algorithm — GitHub, 2023. Actual production source for candidate sourcing, light/heavy rankers, Home Mixer; the-algorithm-ml holds the MaskNet heavy ranker and engagement weights. Primary (code). https://github.com/twitter/the-algorithm
- "Twitter's For You Recommendation Algorithm" — Sumit Kumar (reachsumit.com), 2023. Careful third-party walkthrough of the open-sourced pipeline with component names (RealGraph, GraphJet, SimClusters, TwHIN) and weights. Secondary. https://blog.reachsumit.com/posts/2023/04/the-twitter-ml-algo/
- "The Infrastructure Behind Twitter: Scale" — Twitter Engineering Blog, 2017. Datacenter/hardware/Mesos view of the platform. Primary. https://blog.twitter.com/engineering/en_us/topics/infrastructure/2017/the-infrastructure-behind-twitter-scale
