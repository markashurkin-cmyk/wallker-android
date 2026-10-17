# Walker Wallet 0.2.25 — migration rehearsal and release handoff

**Status: review branch. Production publication is still paused.** No production data or Cloudflare credentials were available for this rehearsal. A real database snapshot has not been tested. Do not treat a synthetic fixture or the visual preview as a restored copy of production.

## What is ready

`npm run test:migration` uses disposable Miniflare/workerd D1 databases and synthetic legacy layouts. It starts once without migration markers and once at migration 21. These fixtures exercise the old table/column shapes; they are not asserted to be an exact dump of a particular deployed version.

The suite compares every row across 37 protected tables: account identities, daily steps, paired device hashes, old campaign rewards, clans, friends, manual grants, custom reward settings/history, daily limits, level claims, Shop balances/orders/purchases/boosts, tap ledgers/receipts, support history, analytics and accumulated rank progress. It includes fractional stored awards and Unicode/quotes. Derived Growth counters are allowed to appear; historical reward rows are not recalculated.

Six scenarios cover:

1. A populated legacy layout gains migrations 21/22 with the protected rows unchanged. Re-running initialization leaves the database unchanged; bounded Growth backfill is repeatable.
2. Migration 22 from an existing marker 21 preserves rows. A fixture snapshot of table SQL, index/trigger definitions, rows and AUTOINCREMENT sequences is restored into a second disposable D1 database and compared. A deleted Founder sequence number is not reused. This is an internal fixture snapshot, not validation of Cloudflare's SQL export format or a production restore.
3. An injected failure after Growth DDL but before settings initialization can be retried. Custom owner Growth criteria survive subsequent starts.
4. Missing live reward settings recover from their saved revision/history instead of factory defaults.
5. Two independent database handles are held at a barrier after both read the old campaign columns. Both cold starts then finish successfully without duplicate-column errors.
6. An unrelated storage error is surfaced, does not mark the migration complete, and a later startup retries safely.

The race in case 5 reproduced a real failure in 0.2.24: `duplicate column name: title_en`. Version 0.2.25 tolerates only that exact duplicate-column race after verifying the resulting column is the expected non-null TEXT with an empty-string default. Other errors still fail the migration. No new migration number, reward rule or UI feature is introduced.

## Before a production decision

Keep the reviewed server ZIP, final commit ID, its checks, the current production deployment/version identifier and an independent backup together. A worker rollback alone is not a database backup. Keep backups private; they contain user IDs, device credentials/hashes, messages and payment history. Never commit a real SQL dump to this public repository.

Obtain a fresh production export and current D1 Time Travel bookmark through the existing authorized Cloudflare account. The installed Wrangler 4.92.0 help confirms these command shapes (they were inspected, **not executed remotely**):

```sh
npx wrangler d1 export wallker-wallet --remote --output=walker-before-growth.sql
npx wrangler d1 time-travel info wallker-wallet --json
```

Check the selected account, database name and ID against the existing production binding before running either command. Record the export's SHA-256, creation time and bookmark outside the public repository. These are read/export steps; do not use a database create/reset command. The SQL export must be restored and rehearsed before relying on it as a backup.

For real-data rehearsal, use a separately authorized private staging database. Sanitize account names, usernames, support text and secret-bearing values in that private copy while preserving IDs/relations, settings revisions, ledger amounts and receipt uniqueness. Disable bot setup, real payments, webhook delivery and external support notifications. Compare schema, row counts, financial sums by source/account and settings revisions before/after migration; repeat the migration and backfill; verify oldest history, paired-device recovery and Shop refunds using synthetic receipts only.

The existing `/growth-preview` is a static sample UI; `GROWTH_PREVIEW=true` intentionally rejects account APIs and disables cron. Do not remove its isolation guard to run a load test. Remote throughput/cost work needs a separate controlled staging environment with no production bot token or payment traffic. Local 100k-account and concurrent-wave tests are not remote capacity guarantees. Ranking refresh writes and retention costs remain described in LOAD_TEST_0.2.23.md.

## Deployment and observation

After real-snapshot rehearsal and the production decision, use the existing deployment path and preserved database binding. Schema creation is lazy on the first API request. First verify the deployed version through health without losing the prior deployment ID; then run an owner-authenticated wallet read to initialize D1 and inspect migrations. Confirm custom tap/step rates, daily caps, balances and connected devices on existing accounts. Observe server errors, migrations, sync/tap replays and payment reconciliation. Expand traffic only after those checks pass.

## Recovery

If the new worker has startup errors, stop further rollout and use the recorded previous worker deployment where compatible with the additive schema. Preserve the database and evidence. Do not delete migration markers, Growth tables, receipts or reward settings to force a retry; initialization is designed to resume.

Do not blindly restore an older database after new walking rewards, purchases, refunds or support messages have arrived. That would discard later activity and could make payment receipts replayable. A database rewind requires a controlled write pause, the verified bookmark/export, a record of all later financial events and a reconciliation plan. Keep ledger receipts and original-bot payment routing intact. No automatic database restore or production rollback is performed by this change.
