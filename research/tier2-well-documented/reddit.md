# Reddit

## One-line hook
Reddit taught the internet to "precompute everything": its front page is not a query, it's a cached answer — every listing in every sort order is computed ahead of time, and even the vote counts you see are deliberately fuzzed lies.

## The core problem
Serve heavily personalized, constantly re-sorted listings (front pages, subreddit pages, comment threads) to hundreds of millions of monthly visitors, on a site where the ranking of every item changes with every vote. Votes arrive in floods (and bots try to inject fake ones), comment threads on big events grow to hundreds of thousands of nested comments, and the tiny team can't afford exotic infrastructure. Reddit's answer was aggressive denormalization: store the same data a hundred ways, precompute every ordering, and make reads dumb and fast while queues absorb the write chaos.

## Architecture overview
End-to-end data flow (classic architecture, per the open-sourced reddit code, Steve Huffman's 2010 talk, and Neil Williams' QCon SF 2017 talk):

1. **Write path (a vote or comment)**: Request hits the CDN/load balancer → a monolithic Python app (Pylons; "r2") validates it → the raw change is written and the expensive consequences (re-ranking listings, updating comment trees, spam checks, karma) are pushed onto RabbitMQ queues → the user gets an immediate response.
2. **The "thing" model**: Nearly every entity — link, comment, account, subreddit — is a "thing" stored in PostgreSQL as two tables: a `thing` table (id, ups, downs, created, deleted flags) and a `data` table of (thing_id, key, value) rows. No joins, no schema migrations to add a property, easy to shard by thing type. Cassandra was layered in (from ~2010) as the key-value store for the ever-growing denormalized data (precomputed listings, comment trees), with memcached in front of everything.
3. **Precomputed listings**: When a link is submitted or voted on, offline job queues recompute every listing that link belongs to, in every sort order (hot, new, top by hour/day/week/…, controversial) — ~15 sort orders, roughly 100 cached variants per item — and store the resulting ID lists in the cache/Cassandra. Rendering a page = fetch a precomputed ID list, hydrate the things, render.
4. **Comment trees**: Comment threads are stored as parent→children structures precomputed per thread and per sort, so rendering a 50,000-comment thread doesn't walk the graph at read time. Updates flow through vote/comment queues sharded by subreddit (with ZooKeeper locks against contention); huge event threads get special handling because queue lag makes trees go stale or inconsistent.
5. **Ranking math (open source)**: The "hot" score for links is `log10(max(1, |ups-downs|)) + sign(ups-downs) * (seconds_since_2005-12-08) / 45000` — vote counts count logarithmically (first 10 votes = next 100 = next 1,000) while age adds a linearly growing bonus so new content can always beat old. Comment sorting uses the Wilson score confidence interval ("best" sort, introduced 2009).
6. **Display path**: The score you see is *fuzzed* — reddit perturbs displayed vote numbers so vote-bot operators can't tell whether their fake votes were silently discarded; the true score still drives ranking.

Component list (plain text):
- CDN (Fastly) + load balancers (HAProxy)
- Monolithic Python app "r2" (Pylons), later surrounded by services
- PostgreSQL "ThingDB": thing table + data table (EAV key-value)
- Cassandra: denormalized listings, comment trees, growing set of key-value data
- memcached (many roles: page cache, render cache, memoization, rate limits; earlier memcachedb for persistence)
- RabbitMQ queues: vote queues, comment-tree queues, spam/thumbnail/award jobs
- ZooKeeper (queue partitioning locks)
- Precompute workers ("mr. jobs") regenerating listings per sort order
- Search (initially Solr/pysolr, later externalized)

## Signature ideas
- **Two tables for everything (the "thing" model).** Reddit abandoned normalized schemas: a generic thing table plus a key-value data table per type. Adding a feature never requires ALTER TABLE on 10M rows; consistency is enforced in application code; there are no joins so data splits across machines trivially. Famously summarized as "reddit's database has two tables."
- **Precompute everything, store it redundantly.** Every listing is materialized in every sort order ahead of time — "wasting disk and memory is better than making users wait." A single link can exist in ~100 cached forms. Reads become key lookups; all ranking work happens in offline queues.
- **The open-source "hot" algorithm.** Because reddit's code was open-sourced (2008), its ranking math became the most-studied feed formula on the internet: logarithmic vote weighting plus a 45,000-second (12.5-hour) time ladder anchored to reddit's 2005 epoch, so a story needs ~10x the votes to beat one posted 12.5 hours later. The "best" comment sort (2009) applied the Wilson score interval — ranking by statistical confidence in the upvote ratio, not raw score.
- **Precomputed comment trees fed by queues.** Comment trees are denormalized structures rebuilt asynchronously; queues are sharded by subreddit with ZooKeeper locks. The failure modes are instructive: processing comment events out of order breaks the tree, and megathreads (elections, AMAs) lag the queue — reddit built a "fastlane" for hot threads.
- **Vote fuzzing.** Displayed up/down counts are intentionally noisy. Spammers who buy 200 bot votes can't distinguish "votes discarded by anti-cheat" from "votes counted then fuzzed," which poisons their feedback loop. A rare example of deliberately serving wrong numbers as a security mechanism (reddit reduced score fuzzing in a 2016 change, acknowledging how much it had distorted displayed scores).
- **r/place (2017): a shared canvas as a systems lesson.** One 1000x1000 pixel canvas, millions of users, one pixel per user per 5 minutes. State = a Redis bitfield of 1M 4-bit color values (~500KB): reads are one bitfield fetch cached by the CDN; a pixel write is a BITFIELD SET plus a RabbitMQ fanout to WebSocket servers broadcasting deltas to every viewer. A masterclass in choosing the right data structure.

## Key numbers
- 7.5M monthly users, 270M pageviews/month, 20+ database servers, ~15 precomputed sort orders, ~100 cached versions per link — 2010, HighScalability writeup of Steve Huffman's lessons: https://highscalability.com/7-lessons-learned-while-building-reddit-to-270-million-page/
- Hot formula constants: log10 of net votes, epoch 1134028003 (Dec 8, 2005), divisor 45000 seconds (12.5 hours) — from reddit's open-sourced code, explained in "How Reddit ranking algorithms work" (Amir Salihefendic, 2010/2015): https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9
- 6 billion pageviews/month era and growth pains — 2013, HighScalability "Reddit: Lessons Learned from Mistakes Made Scaling to 1 Billion Pageviews a Month": https://highscalability.com/reddit-lessons-learned-from-mistakes-made-scaling-to-1-billi/
- r/place: 1000x1000 canvas as 1M x 4-bit Redis bitfield (~4Mb), >1M users placed ~16M+ pixels over 72 hours, ~90K peak concurrent viewers/editors — 2017, reddit blog "How We Built r/Place" and RedisConf17 talk: https://redditblog.com/2017/04/13/how-we-built-rplace/ and https://www.slideshare.net/RedisLabs/redisconf17-reddit-how-we-built-and-scaled-rplace
- Vote/comment queues sharded by subreddit with ZooKeeper locks; comment trees denormalized into Cassandra/cache — 2017, Neil Williams, "The Evolution of Reddit.com's Architecture," QCon SF: https://www.infoq.com/presentations/reddit-architecture-evolution/ (slides: https://qconsf.com/sf2017/system/files/presentation-slides/qconsf-20171113-reddits-architecture.pdf)
- Reddit ~330M MAU at the time of the 2017 QCon talk — 2017, QCon SF presentation page: https://archive.qconsf.com/sf2017/presentation/evolution-redditcoms-architecture

## Evolution timeline
- **2005**: Launch (Lisp, quickly rewritten in Python); single Postgres.
- **2008**: reddit open-sources its codebase — ranking algorithms become public knowledge.
- **2009**: "Best" comment sort ships (Wilson score interval, blog post co-authored with Randall Munroe's endorsement of the math).
- **2010**: Steve Huffman's scaling lessons: thing model, precompute everything, memcached/memcachedb, RabbitMQ offline work; Cassandra introduced after memcachedb falters.
- **2011-2015**: Move to AWS-era maturity; Cassandra absorbs more denormalized data; monolith "r2" persists with queues and cache layers around it.
- **2016**: Vote-score display change — reddit admits and reduces years of score fuzzing distortion.
- **2017**: r/place; QCon talk describes breaking the monolith into services, sharded vote queues, comment-tree fastlane.
- **2018+**: New frontend and API-driven redesign; GraphQL federation and service decomposition (later blog posts); open-source repo archived (reddit-archive/reddit).

## Visualization hooks
- The two-table universe: every reddit concept (user, post, comment, subreddit, award) collapsing into just [thing] and [data] tables — a zoo of shapes funneled into two boxes.
- "Precompute everything": one incoming vote triggering a fan of ~15 sort-order recomputations and ~100 cached copies lighting up; the read path as a single arrow into a cache.
- The hot algorithm as a staircase: log10(votes) curve vs the time ladder that adds +1 every 12.5 hours; show a 10-vote new post outranking a 100-vote half-day-old post.
- A comment megathread as a growing tree with queue workers grafting branches on; a lagging queue shown as branches appearing out of order / broken.
- Vote fuzzing: a spammer's dashboard where the numbers wobble every refresh — true score ledger hidden behind a noisy display counter.
- r/place: the 1M-pixel canvas as one long Redis bitfield ribbon; a single pixel write rippling out through RabbitMQ to thousands of WebSocket clients; the CDN serving the full canvas snapshot.
- Wilson score "best" sort: two comments — 5/5 upvotes vs 90/110 — and the confidence-interval bars explaining why the second wins.

## Sources
- "7 Lessons Learned While Building Reddit to 270 Million Page Views a Month" — HighScalability, 2010. Writeup of Steve Huffman's talk: thing model, precompute-everything, memcache(db), queues. Secondary (faithful to primary talk). https://highscalability.com/7-lessons-learned-while-building-reddit-to-270-million-page/
- "Reddit's database has two tables" — Kevin Burke, 2012. Quotes Huffman on the thing/data schema rationale (schema changes, no joins). Secondary. https://kevin.burke.dev/kevin/reddits-database-has-two-tables/
- Architecture Overview — reddit-archive/reddit GitHub wiki (written by reddit engineers while the code was open source). App tier, ThingDB, Cassandra, memcached, RabbitMQ. Primary. https://github.com/reddit-archive/reddit/wiki/architecture-overview
- reddit-archive/reddit — the open-sourced codebase itself (2008-2017), including `_sorts.pyx` with the hot/controversial/confidence functions. Primary (code). https://github.com/reddit-archive/reddit
- "How Reddit ranking algorithms work" — Amir Salihefendic (Hacking and Gonzo, Medium), 2010 (updated 2015). The standard walkthrough of the hot formula and Wilson "best" sort with the actual open-source code. Secondary (code-derived). https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9
- "reddit's new comment sorting system" — reddit blog, 2009. Announces Wilson-score "best" sorting for comments. Primary. https://redditblog.com/2009/10/15/reddits-new-comment-sorting-system/
- "The Evolution of Reddit.com's Architecture" — Neil Williams, QCon SF 2017 (InfoQ video + slides). Monolith history, sharded vote/comment queues, ZooKeeper locks, comment-tree pitfalls, fastlane. Primary (talk). https://www.infoq.com/presentations/reddit-architecture-evolution/
- "How We Built r/Place" — reddit blog (engineering), April 2017. Full design: Redis bitfield state, CDN-cached canvas, RabbitMQ + WebSocket fanout, rate limiting. Primary. https://redditblog.com/2017/04/13/how-we-built-rplace/
- "RedisConf17 — How We Built and Scaled r/place" — reddit engineers' conference deck, 2017. BITFIELD commands and scaling specifics. Primary (talk). https://www.slideshare.net/RedisLabs/redisconf17-reddit-how-we-built-and-scaled-rplace
- "Reddit: Lessons Learned from Mistakes Made Scaling to 1 Billion Pageviews a Month" — HighScalability, 2013. Later-era scaling retrospective. Secondary. https://highscalability.com/reddit-lessons-learned-from-mistakes-made-scaling-to-1-billi/
- On vote fuzzing: reddit's historical FAQ and the 2016 admin post on vote-score display changes; treat precise fuzzing mechanics as intentionally undocumented (anti-abuse). Primary-ish but deliberately vague. https://www.reddit.com/r/announcements/comments/5gvd6b/scores_on_posts_are_about_to_start_going_up/
