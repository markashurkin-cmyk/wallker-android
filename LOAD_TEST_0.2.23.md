# Walker Wallet 0.2.23 — 100,000-account local benchmark

Executed 2026-10-16 against disposable local Miniflare/workerd D1. All accounts, devices, steps, referrals, clans and activity rows are synthetic. No production data, token, payment or remote D1 connection is used.

Dataset: 100,000 accounts; 10,150 inviters; 89,850 qualified and Active invitees; 150 Creator-eligible inviters with 500 referrals each; 100 clans of 100 members; one active referral/Creator season; mature D7 and D30 cohorts.

The initial test exposed repeated eligibility counts inside the Creator season query: both counts ran again for every active invitee. Version 0.2.23 groups eligibility once per inviter and calculates lifetime-at-cutoff Active and season score in a single pass. Frozen criteria and exact [start,end) timing remain intact. No schema migration or reward recomputation is added.

| Operation | Before, local ms | After, local ms | Before reported reads | After reported reads |
| --- | ---: | ---: | ---: | ---: |
| Rank refresh including Creator season | 16,591.7 | 405.7 | 76,755,168 | 1,581,669 |
| Season finalization | 16,750.7 | 246.7 | 76,293,805 | 1,120,306 |
| Already-initialized backfill, 80 accounts | 51.7 | 32.7 | 40,402 | 403 |

Creator analytics now refreshes up to 20 accounts with five D1 round trips, compared with 37 trips for five accounts before. The measured batch took 115.1 ms; mature synthetic cohorts retain their exact D7=50% and D30=25% results. Zero/immature cohort denominators remain unavailable rather than zero retention.

| Warm operation, five sequential samples | Local median ms | D1 round trips per request |
| --- | ---: | ---: |
| Wallet | 255.3 | 33 |
| Tap | 229.6 | 25 |
| My Pack | 129.1 | 20 |

All four leaderboards and the seasonal own-position read completed locally in 48–60 ms in this run. A user ranked below TOP 1000 still receives their own exact position. A repeated finalized-season job writes no duplicate result or badge. Forty-two additional cases compare database scores with an independent JavaScript oracle: frozen thresholds, start/end boundaries, Moscow midnight, missing activations, legacy activity, and restrictions/freezes on either side.

## Reproduction and regression guards

Run `npm run test:load` for the synthetic benchmark. It also runs at the end of `npm test`, so the existing Chromium/WebKit CI workflow verifies the same server path. Output is `growth-load-results.json`; detailed aggregate counters are printed to the job log.

Guards enforce fewer than 3M reported rank-refresh reads, fewer than 2M finalization reads, fewer than 2,000 initialized-backfill reads, at most six Creator analytics round trips, and bounded wallet/tap/My Pack requests. They deliberately avoid fragile runner-specific timing thresholds.

## Limits before public scale

These are local sequential timings, not Cloudflare edge p95, real-device concurrency, a billing quote or proof that 100,000 people can tap simultaneously. Reported row counters cover operations returning D1 metadata; `first()` does not expose row metadata and its reads are not included. Round-trip counts include all statement calls and transactional batches, including telemetry for API tests.

A snapshot contains 30,850 rank rows on this dataset. Refreshing every 15 minutes inserts about 2,961,600 rank records per day before index amplification and pruning. Observed snapshot writes including indexes were 92,554. The query improvement reduces reads; it does not eliminate the snapshot write cost. Remote staging must establish query limits, actual row accounting, CPU/latency, daily budget and concurrent-device behavior; adjust refresh cadence or move to incremental ranking storage before broad traffic if those limits require it.

Production publication remains paused. This change is confined to the review branch and its isolated preview. Final commit CI and Cloudflare preview status must be checked separately from these local measurements.
