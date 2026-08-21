# SellToner Price Freshness Monitor — Design

**Status: BLOCKED on recon. Do not start Phase 1.**

Phases 1–4 below assume four facts nobody has verified: whether `easyAdd()` posts to the server, whether it writes rows into SellToner's database, what the auth token's real TTL is, and whether a bulk endpoint exists. RECON.md in this folder answers all four. Run it first — the answers change the architecture, and one of them (server-side record creation) can invalidate this approach entirely.

First Claude Code session should be: work through RECON.md, then rewrite Part 2 and Part 3 of this document with the real endpoints, payloads, and token lifetime. Only then plan Phase 1.

Reference material already on disk: GOOGLE-SHEET-SCRAPE-PLAYBOOK.md in this project, and the `selltoner-buyback-scrape` skill. The local `scraper.py` writes a CSV — a different pipeline, but read its HTTP handling before writing the new collector.

## Context

You refresh SellToner buyback prices via the `selltoner-buyback-scrape` skill: Claude drives Chrome, walks an AngularJS scope on `app.selltoner.com/#/vendor-offer/create`, prices ~1,259 SKUs, and pastes results into the SELLTONER-SUPPLYLINK Google Sheet. It's a once-daily full-catalog batch fighting a 5-minute token expiry, 45s CDP timeouts, and `window.name` data smuggling.

Your constraints, now that they're pinned down:

- Prices feed repricing code that quotes customers. Stale prices cost conversions.
- A price up to 4 hours old is fine. Past 4 hours it is not.
- Account but no rep — stay quiet, no asking SellToner for a feed.
- Hybrid: Chrome for session, direct API for queries.
- Tier membership can be built from real volume/margin data.
- Hosting: your call to me.

## Part 1 — The 4-hour answer changes the problem

I was designing for near-real-time. You don't need it, and saying so is the most useful thing in this document.

### This is a freshness SLA, not a latency target

"Near-real-time" means detect changes fast. "Nothing older than 4 hours" means guarantee an upper bound on age. Those sound similar and produce very different systems:

|  | Latency target | Freshness SLA |
| --- | --- | --- |
| Optimizes for | speed of detection | absence of stale data |
| Fails when | a change is detected slowly | a sweep doesn't run |
| Engineering is | scheduling cleverness, tiering, change-signal probes | reliability — sessions, retries, alerting |
| Top-line metric | detection latency p95 | max age of any quotable SKU |

Your risk isn't that a price change takes 40 minutes to notice. It's that the session silently expires on a Tuesday and you quote from 30-hour-old data for a day and a half. The whole effort should move from "poll faster" to "never miss a cycle, and scream loudly when you do."

Concretely, this deletes: 15-minute Tier A polling, adaptive tier promotion/demotion, the change-signal probe hunt as a prerequisite, and most of the ban risk that came with high-frequency access. A 4-hour ceiling is satisfied by 6–8 scheduled refreshes a day. That's an unremarkable traffic pattern for a real vendor account.

### Conversion loss is asymmetric — and that's actionable

You lose conversions when you quote a customer too low. That happens when SellToner's buyback price went up and you didn't know. The reverse (price dropped, you quote high) doesn't cost conversions — you win that deal and lose margin instead. Different problem, different urgency.

So: treat detected price increases as the higher-priority signal. Practically —

- A confirmed increase publishes immediately and, if large, pings you.
- A confirmed decrease can ride the normal cycle.
- A price → null on a SKU is dangerous specifically because your repricing code may fall back to a stale or default value and under-quote. Nulls need the confirmation gate more than anything else does.

### One honest caveat about your cost metric

Conversion loss is real but nearly unmeasurable — you never meet the customer who quietly went elsewhere. That means you will not be able to A/B this system into justifying itself, and you'll be tempted to keep spending on precision you can't observe returns from. The 4-hour number is the discipline: hit it reliably, then stop. Don't let "conversions" become an open-ended justification for chasing minutes.

## Part 2 — Phase 0: Recon (still first, but smaller)

Two questions matter now; two dropped in importance because you don't need high frequency.

**Q1 — Does `easyAdd()` create server-side records?** (now the critical one) The skill's "reset the offer between chunks" hints state accumulates. If every price check writes a draft offer row into SellToner's database, then even 7 sweeps/day leaves ~9,000 junk rows a day in their admin — trivially visible and the fastest route to getting shut off. Answer this before anything else. If yes, the pricing path must move to a client-side formula (Q2) or the frequency must drop.

**Q2 — Does `easyAdd()` compute client-side, or POST?** Network tab, one `easyAdd()`, watch. Client-side means one cheap GET per SKU and a formula you can replicate offline — halving all traffic and defusing Q1 entirely.

**Q3 — What is the auth token, really?** JWT in localStorage? Actual `exp`? Refresh endpoint? Under a freshness SLA this is a reliability question, not a convenience one — a token that dies unnoticed is exactly the failure that breaches the 4-hour bound. Solving refresh properly retires skill constraint #3.

**Q4 — Cheap change signal (`updatedAt`, ETag, bulk endpoint)?** Still worth ten minutes of looking, but it's now an optimization rather than a prerequisite. Nice if it exists; the plan works without it.

## Part 3 — Architecture

```
┌─ SESSION SVC ─┐   ┌─ COLLECTOR ─────────┐   ┌─ CONFIRM GATE ─┐   ┌─ STORE ──┐
│ Playwright    │──▶│ 2 tiers, scheduled   │──▶│ 2-of-3 before  │──▶│ prices + │
│ OAuth login   │   │ direct-API price call│   │ publishing     │   │ history  │
│ token refresh │   │ budget + breaker     │   │ null ≠ change  │   │ + age    │
└───────────────┘   └──────────────────────┘   └────────────────┘   └────┬─────┘
        ▲                                                                │
        └──── SLA WATCHDOG: max age > 3.5h ⇒ page James ────┐            │ price.changed
                                                            │  ┌─────────┼─────────┐
                                                            ▼  ▼         ▼         ▼
                                                          Slack    Repricing   Sheet
                                                          alert    engine      writer
```

**1. Session service.** Headless Playwright does the Google OAuth login once, exports cookie/JWT to a token store; a refresher keeps it warm per Q3. Hard expiry is a paging event, not a log line — under an SLA, a dead session is an outage.

**2. Two tiers.** That's all you need.

| Tier | Membership | Refresh | Why |
| --- | --- | --- | --- |
| Quotable | SKUs customers actually get quoted on, from your volume data | every 3 h | 3h refresh gives ~1h of buffer under the 4h ceiling for retries and slow runs |
| Long tail | Everything else + the ~102 known not-founds | nightly | Never quoted ⇒ no conversion exposure ⇒ no SLA |

No adaptive promotion, no change-rate heuristics. A SKU is either quotable or it isn't, and that comes from data you already have. Recompute membership monthly from the volume export; anything newly quoted defaults into the quotable tier until proven otherwise — the safe direction to be wrong.

Load (assuming `easyAdd` POSTs, so 2 req/SKU; halve if Q2 says client-side):

| Approach | Requests/day | Meets 4h SLA? |
| --- | --- | --- |
| Today — one daily sweep | ~2,500 | ✗ (up to 24h stale) |
| Full catalog every 3h | ~17,600 | ✓ |
| Quotable-tier every 3h + nightly full | ~5,300 | ✓ where it matters |
| (my earlier near-real-time plan) | ~12,700 | ✓, at 2.4x the cost, for freshness you don't use |

~5,300/day is roughly 2x your current footprint — a defensible traffic increase for a real vendor account, and a fraction of what near-real-time would have cost.

**3. Confirmation gate.** No event from a single observation. A candidate change reconfirms on an independent request before publishing. `price → null` is suspect, never "delisted" — quarantine and escalate. That's the 67-false-nulls lesson as a rule. Bounds: a >50% or >$40 swing goes to human review before it touches quotes. Increases publish on the fast path; decreases ride the normal cycle.

**4. SLA watchdog** — the component this design is really about. A job that does nothing but ask: what is the oldest quotable-SKU price right now? If it crosses 3.5 hours, alert before the SLA is actually breached. This catches every failure mode that matters — dead session, blocked account, crashed scheduler, machine asleep — and it's the one dashboard number you should actually watch.

**5. Store + events.** `prices` (current, with `observed_at`) + `price_history` (append-only, every observation including unchanged reads — that's what makes staleness and change rate measurable). On a confirmed delta, emit `price.changed` with old/new/delta/direction/confidence/`observed_at`. HMAC-signed, idempotency key, retry with backoff. (Add a `vendor` column now — ten minutes today versus a migration on a populated history table later.)

**6. Repricing engine** subscribes over HTTP. Two hard requirements on its side: idempotent (at-least-once delivery means duplicates will arrive) and stale-event rejection by comparing `observed_at` to what it last applied. Third, specific to your SLA: it should refuse to quote from any price older than 4 hours rather than trusting the pipeline to have kept it fresh. Enforce the bound at the point of use, not just at the point of collection.

**7. Sheet becomes a subscriber, not the database.** Written incrementally via the Sheets API. This retires the "anchor C2, screenshot to confirm, dispatch a synthetic ClipboardEvent" ritual — currently the most dangerous step in your pipeline, since a wrong anchor overwrites your MODEL column.

**8. Politeness.** Concurrency ≤3, jitter, hard requests/hour ceiling, circuit breaker on 429/403/captcha/latency inflation. Alert on a null-rate spike — that means blocked-or-broken, not "prices vanished." Kill switch reachable from your phone. At 7 sweeps/day you're not straining them; the breaker is there to protect you from a runaway bug.

**Stack:** Python + httpx/asyncio + Playwright (session only) + SQLite + APScheduler + a small FastAPI emitter. Same language as your existing `scraper.py`.

**Hosting:** leave your office machine on 24/7 (`pmset`/`caffeinate`). IP consistency is the biggest anti-ban lever for an authenticated account — a fresh datacenter IP suddenly driving your session is exactly the anomaly that flags accounts. Only the dashboard/alert receiver goes on a VPS, with no SellToner traffic from it. This is now an SLA requirement, not a preference: the weekend shutdown your `sku-audit-offers-to-sheet` skill documents is a guaranteed 48-hour breach every week.

## Part 4 — Build order

| Phase | Deliverable | Payoff |
| --- | --- | --- |
| 0 | Recon Q1–Q3 + log every observation to `price_history` | Answers whether the pricing path is safe to run 7x/day |
| 1 | Session service + direct-API client replacing the browser scrape | Sweeps run unattended in minutes — the precondition for any SLA |
| 2 | Price store + history + nightly full sweep as system of record | Sheet becomes an output, not the database |
| 3 | Quotable tier every 3h + SLA watchdog | The 4-hour guarantee, delivered |
| 4 | Confirmation gate + `price.changed` into repricing | Automation that can't be poisoned by a transient null |

Phase 3 is the one that satisfies your actual requirement. Phase 4 is what makes it safe to let run unsupervised — do not point the repricing engine at live events before the gate exists.

## Verification

- **Phase 0:** written answers to Q1–Q3 backed by a HAR capture. Q1 in particular gets an explicit yes/no before Phase 3 is scheduled.
- **Phase 1:** the direct-API sweep must reproduce the 2026-06-15 baseline — 1,259 models, 1,157 priced, column-C sum $49,465.00, ~102 not-found. Deviation means a port bug, not a price change.
- **Phase 3:** run for two weeks and assert max quotable-SKU age never exceeded 4 hours, from the history table, not from vibes. Then deliberately kill the session mid-week and confirm the watchdog pages you before the bound is breached. An SLA you haven't tested failing is not an SLA.
- **Phase 4:** replay historical `price_history` through the gate and verify zero events fire for the 67 known transient nulls. Deliver one event twice; confirm the second application is a no-op.
- **Ongoing dashboard:** max quotable-SKU age (the headline number), sweeps completed vs scheduled, 401 rate, null rate, change rate by direction, requests/hr.

## Open question

Roughly how many SKUs are actually quotable? It sets the entire load figure — I've assumed ~200 of your 1,259. If it's closer to 800, the "quotable every 3h" and "full catalog every 3h" options converge and you should just sweep everything on the 3-hour cycle and skip tiering altogether. Your volume data answers this in about five minutes, and it's the last thing standing between this and a build.
