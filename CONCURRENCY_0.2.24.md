# Walker Wallet 0.2.24 — concurrent request validation

Production publication remains paused. This test never contacts a live bot, account or D1 database.

## Method

`npm run test:concurrency` launches Promise.all waves of signed HTTP requests against Miniflare/workerd and a disposable real SQLite/D1 binding. The driver has up to 100 requests in flight; Workers/D1 can queue execution. Local timings do **not** establish remote throughput, production p95, 100,000 simultaneous users, or a Cloudflare capacity guarantee.

## Observed local run

Created: 2026-10-17T04:51:57.793Z.

| Wave | Requests | Expected HTTP | Driver in flight | Local duration (ms) |
| --- | ---: | ---: | ---: | ---: |
| 100-players-tap | 100 | 200 | 100 | 2583 |
| 24-identical-retries-one-account | 24 | 200 | 24 | 532 |
| 24-competing-cap-claims-one-account | 24 | 200 | 24 | 443 |
| 24-overlapping-step-reports | 24 | 200 | 24 | 467 |
| 50-first-open-referrals | 50 | 200 | 50 | 1647 |
| 20-competing-inviters-one-new-account | 20 | 200 | 20 | 577 |
| 100-players-read-my-pack | 100 | 200 | 100 | 1172 |
| 20-unauthenticated-tap-requests | 20 | 401 | 20 | 39 |

All 362 measured requests returned their expected status. Warmup and follow-up assertions are additional requests.

## Accounting invariants

- Replay exactly once; competing requests stop at exact 3 WC cap; energy deducted once; other accounts unchanged.
- Out-of-order repeated step reports converge to max 12000 steps and exactly 4000 WC, with rank progress capped at 10000/day and without double credit.
- 51 distinct new accounts counted once; competing inviters cannot duplicate attribution; no fixed invite coin awards or retroactive tap changes.
- Achievement deduplicated under read waves; unauthenticated traffic creates no rewards.

The sync fixture deliberately submits 1,000–12,000 steps out of order. The accepted day total is 12,000; daily trusted rank progress is capped at 10,000 by the existing policy. The rank cap is unchanged. A test expectation was corrected to distinguish the two totals.

## Remaining validation

A separately authorized staging endpoint with real D1 and controlled synthetic traffic is still required to measure remote throughput and cost. The visual preview rejects real account APIs and cannot be used for that test. No production load test, migration or deployment was performed.
