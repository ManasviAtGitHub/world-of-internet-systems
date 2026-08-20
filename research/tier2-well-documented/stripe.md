# Stripe

## One-line hook
How do you charge a credit card *exactly once* over a network that only promises *at least once*?
Stripe's answer — the idempotency key — became the industry's canonical pattern for making money
movement as safe as a database transaction.

## The core problem
Payments are the worst possible workload for a distributed system: every request moves real money,
every failure mode is visible on someone's bank statement, and the network between merchant, Stripe,
card network, and issuing bank can fail at any point. A timeout is ambiguous — did the charge happen
or not? Retrying naively double-charges; not retrying loses revenue. Stripe has to solve
exactly-once *semantics* (not delivery, which is impossible) at enormous scale: $1.4 trillion of
payment volume in 2024, roughly 1.3% of global GDP, while keeping an API stable enough that an
integration written in 2011 still works unchanged today.

## Architecture overview

**End-to-end journey of one card charge:**

1. **Merchant server → Stripe API.** The merchant's backend calls `POST /v1/payment_intents` (or
   `/charges`) with an `Idempotency-Key` header and its API key. Card numbers never touch the
   merchant's server in a modern integration — the browser/mobile SDK tokenizes the card directly
   against Stripe's PCI-scoped vault, so the merchant only handles an opaque token.
2. **API edge.** The request passes Stripe's rate limiters and load shedders (four layers — see
   below), gets pinned to the account's API version, and is checked against the idempotency-key
   store: if this key was already completed, the stored response is returned immediately and nothing
   re-executes.
3. **Risk scoring.** Stripe Radar scores the transaction with ML models (fraud signals aggregated
   across the whole network) in the synchronous path; a blocked charge never reaches the bank.
4. **Authorization leg.** Stripe, acting with its acquiring partners, formats an authorization
   request and sends it over the card network (Visa, Mastercard, etc.) to the **issuing bank** (the
   customer's bank). The issuer checks funds/fraud and returns approve/decline. This round-trip
   through merchant → acquirer → network → issuer and back typically completes in ~1–2 seconds.
5. **Capture, clearing, settlement.** Authorization only places a hold. Capture (immediate or later
   — the classic two-phase authorize/capture pattern) triggers clearing: batched files flow through
   the network, the issuer transfers funds to the acquirer, and Stripe pays out to the merchant's
   bank account on a schedule, minus fees.
6. **Recording.** Every internal money-state transition is emitted as an immutable event into
   **Ledger**, Stripe's double-entry system of record, where automated data-quality checks prove the
   books balance.
7. **Webhooks** notify the merchant asynchronously of state changes (`payment_intent.succeeded`,
   disputes, payouts).

**Component list (plain text):**
- Client SDKs / Elements (tokenization in browser, keeps merchants out of PCI scope)
- API gateway: auth, rate limiters, load shedders, idempotency-key cache, API version pinning
- Core API application: historically one large Ruby monolith, type-checked by Sorbet
- Radar: ML fraud scoring in the hot path
- Payments core: connections to acquirers and card networks (authorize/capture/refund state
  machines)
- Ledger + Data Quality platform: immutable event log, double-entry validation
- Webhook delivery, Sigma/analytics, Billing, Treasury and other products layered on the same rails

## Signature ideas

**Idempotency keys.** The client generates a unique key per logical operation and retries with the
same key on any ambiguous failure; the server stores the result of the first completed execution and
replays it for duplicates. Combined with client-side exponential backoff plus random jitter (to
avoid thundering herds), this converts an unreliable network into effectively-exactly-once money
movement. Brandur Leach's 2017 post is *the* canonical write-up of the pattern and is cited by
practically every payments API since.

**Rolling, date-named API versions.** Instead of v1/v2 big-bang versioning, every
backwards-incompatible change becomes a new version named by release date (e.g. `2017-05-24`). An
account is pinned to the version in force at its first request, forever. Internally, each breaking
change is a "version change module" holding a description and a transformation function; responses
are generated in the newest shape, then transformed backward through the stack of modules until they
match the caller's pinned version. By 2017 this let Stripe ship ~100 breaking changes in six years
while never breaking a customer.

**Four-layer rate limiting and load shedding.** (1) a per-user request rate limiter (token bucket in
Redis), (2) a concurrent-requests limiter for expensive endpoints, (3) a fleet usage shedder that
reserves a slice of capacity (e.g. 20%) for critical methods like charge creation, and (4) a worker
utilization shedder that drops traffic in priority order (test-mode first, then GETs, then POSTs,
never critical methods) as workers saturate. New limiters are "dark launched" to observe what they
*would* block before enforcement.

**The typed Ruby monolith (Sorbet).** Rather than decompose into microservices, Stripe built Sorbet,
a fast static type checker for Ruby, and ran it across ~15 million lines of code in 150,000 files.
Types recovered the navigability and refactoring safety a monolith loses at scale; Stripe
open-sourced Sorbet in 2019 and it's now the standard Ruby type checker. A deliberate
counter-example to "you must split the monolith."

**Ledger: double-entry bookkeeping as a systems-correctness tool.** All internal money-producing
systems are modeled as state machines emitting immutable events; balances move between accounts and
every fund flow must "clear" to zero, exactly like accounting T-accounts. A data-quality platform
continuously measures clearing, timeliness, and completeness, giving mathematical proof-shaped
guarantees (99.9999% "explainability" of money movement) rather than hoping reconciliation scripts
catch drift.

**Authorize/capture as a distributed two-phase commit with the banking system.** The card network's
own auth-hold-then-capture design is effectively a prepare/commit protocol spanning organizations;
Stripe's API surfaces it directly (auth holds, delayed capture, partial capture), a great teaching
example that "two-phase" patterns predate computing fashion.

## Key numbers
- **$1.4 trillion** total payment volume in 2024, up 38% YoY, ≈1.3% of global GDP — Stripe newsroom,
  Feb 27, 2025. https://stripe.com/newsroom/news/stripe-2024-update
- **$31 billion** processed over Black Friday–Cyber Monday 2024; **465M transactions**; peak
  **137,000 transactions/minute**; API uptime **>99.9999%** during the weekend — Stripe newsroom,
  Dec 2024. https://stripe.com/newsroom/news/bfcm2024
- **$40+ billion** processed over BFCM weekend 2025 — Stripe's live BFCM data page, Dec 2025.
  https://bfcm.stripe.com/
- **5 billion events/day** into Ledger; **99.9999% explainability** of money movement; 99.99% of
  dollar volume fully ingested and verified within four days; 135+ currencies, 185 countries —
  Stripe dev blog "Ledger," Feb 2024.
  https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement
- **~15 million lines of Ruby, 150,000 files** type-checked by Sorbet — Stripe blog, 2019
  (open-sourcing announcement era). https://stripe.com/blog/sorbet-stripes-type-checker-for-ruby
- **~100 backwards-incompatible API upgrades in 6 years** with zero broken integrations;
  compatibility maintained with every version since 2011 — Stripe blog "APIs as infrastructure," Aug
  2017. https://stripe.com/blog/api-versioning
- **~200 million active subscriptions** managed by Stripe Billing; used by 300,000+ companies —
  Stripe newsroom, Feb 2025. https://stripe.com/newsroom/news/stripe-2024-update

## Evolution timeline
- **2010–2011** — Founded (as /dev/payments); launches with the famous "seven lines of code"
  integration pitch; API versioning discipline dates from 2011.
- **2017** — The canonical API-craft trilogy published: idempotency keys (Feb), rate limiters (Mar),
  rolling API versioning (Aug). These posts define Stripe's public engineering identity.
- **2017–2019** — Sorbet built internally to save the Ruby monolith; open-sourced June 2019. Stripe
  explicitly delays service decomposition past ~3,000 engineers.
- **~2018–2021** — Product surface explodes (Radar, Billing, Terminal, Treasury, Issuing) on the
  same payments rails; Global Payments and Treasury Network framing appears.
- **2021–2024** — Ledger and the Data Quality platform built out as the financial system of record;
  publicized Feb 2024.
- **2024–2025** — $1.4T volume year; first BFCM tracking crypto payments (2024); >$40B BFCM 2025.
  Uptime marketed at five-to-six nines.

## Visualization hooks
1. **The life of a charge** — a horizontal swimlane: browser → merchant server → Stripe API → Radar
   → card network → issuing bank → back, with the ~1–2s auth round-trip and the days-later
   settlement flow drawn as two separate tempos (fast lane / slow lane).
2. **Idempotency key retry storyboard** — three panels: request lost (retry safe), server died
   mid-work (retry resumes), response lost (retry replays stored result). Same key, three failure
   points, one charge.
3. **Token bucket** — literal bucket with tokens dripping in, requests scooping them out; empty
   bucket = 429.
4. **Load-shedding priority stack** — a pressure gauge pushing traffic classes overboard in order:
   test mode → GETs → POSTs, with "critical methods" bolted to the deck.
5. **Version time machine** — a response object passing backward through a chain of dated
   transformation modules (2024-06-20 → … → 2013-02-11) until it matches the caller's pinned year.
6. **Double-entry T-accounts** — one $100 charge fanning into balancing entries: customer card,
   Stripe fees, network fees, merchant payout — all columns summing to zero.
7. **Monolith with a exoskeleton** — the Ruby monolith drawn as one big organism with Sorbet as a
   typed skeleton, versus the microservices alternative drawn as scattered cells.
8. **BFCM pulse** — 137k transactions/minute peak drawn as a heartbeat spike over the weekend.

## Sources
- **"Designing robust and predictable APIs with idempotency"** — Stripe blog, Brandur Leach, Feb 22,
  2017. The canonical idempotency-key post: retry semantics, backoff+jitter, ACID considerations.
  *Primary.* https://stripe.com/blog/idempotency
- **"APIs as infrastructure: future-proofing Stripe with versioning"** — Stripe blog, Brandur Leach,
  Aug 5, 2017. Rolling date-named versions, account pinning, version change modules. *Primary.*
  https://stripe.com/blog/api-versioning
- **"Scaling your API with rate limiters"** — Stripe blog, Paul Tarjan, Mar 30, 2017. The four
  limiters/shedders, token bucket on Redis, dark launching. *Primary.* (Companion gist with code:
  https://gist.github.com/ptarjan/e38f45f2dfe601419ca3af937fff574d)
  https://stripe.com/blog/rate-limiters
- **"Sorbet: Stripe's type checker for Ruby"** — Stripe blog, 2019 (+ sorbet.org open-sourcing post,
  June 20, 2019). Why types saved the 15M-line monolith. *Primary.*
  https://stripe.com/blog/sorbet-stripes-type-checker-for-ruby ,
  https://sorbet.org/blog/2019/06/20/open-sourcing-sorbet
- **"Ledger: Stripe's system for tracking and validating money movement"** — Stripe dev blog, Ilya
  Ganelin, Feb 16, 2024. Immutable events, state machines, double-entry validation, DQ platform,
  scale numbers. *Primary.*
  https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement
- **Stripe 2024 annual-letter newsroom summary** — Stripe, Feb 27, 2025. $1.4T TPV, growth, product
  stats. *Primary (company-reported).* https://stripe.com/newsroom/news/stripe-2024-update ; full
  letter PDF:
  https://assets.stripeassets.com/fzn2n1nzq965/2pt3yIHthraqR1KwXgr98U/b6301040587a62d5b6ef7b76c904032d/Stripe-annual-letter-2024.pdf
- **"Businesses processed more than $31 billion on Stripe from Black Friday through Cyber Monday"**
  — Stripe newsroom, Dec 2024; and live page https://bfcm.stripe.com/ (2025 edition, $40B+). Peak
  TPS and uptime claims. *Primary (company-reported marketing numbers).*
  https://stripe.com/newsroom/news/bfcm2024
- **"Why did Stripe build Sorbet?"** — Will Larson (lethain.com), 2024. Outside analysis of the
  monolith-preservation strategy (~3,000 engineers before decomposition). *Secondary,
  well-informed.* https://lethain.com/stripe-sorbet/
- **Stripe docs: rate limits, API upgrades, idempotency** — living reference for current behavior.
  *Primary.* https://docs.stripe.com/rate-limits , https://docs.stripe.com/upgrades ,
  https://docs.stripe.com/api/idempotent_requests
- Note on the charge-flow description: authorize/capture, issuer/acquirer/network roles are standard
  payments-industry mechanics documented across Stripe's docs (e.g.,
  https://stripe.com/guides/introduction-to-online-payments) rather than a single engineering post.
