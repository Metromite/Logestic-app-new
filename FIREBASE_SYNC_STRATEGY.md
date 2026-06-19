# Sync Strategy — Staying on Firebase's Free (Spark) Tier

Spark tier limits (Firestore): 50K reads/day, 20K writes/day, 20K deletes/day, 1GiB stored.
This app's design treats those as a hard budget, not a target.

## Core principle: SQLite is the source of truth. Firestore is a mirror.

Every screen reads/writes SQLite directly via Tauri commands (`db.rs`). The UI is never
blocked on network and never queries Firestore directly for normal operation. Firestore
exists purely so a second device/user can eventually catch up.

## Write path (push): outbox + batched commits

1. Every mutation (`run_mutation` in `db.rs`) writes to the operational table **and**
   appends a row to `_sync_log` in the same SQLite transaction — this is a local
   write-ahead outbox, identical in spirit to the original Python `_sync_log` table.
2. `sync_worker.rs` runs on a 45s timer (configurable), pulls up to 200 pending outbox
   rows, and sends them as a **single** Firestore `:commit` REST call
   (`writes` array) — collapsing what would be N writes-in-N-requests into 1 request.
   This matters less for the 20K/day write quota itself (each document write still
   counts individually) but is critical for staying within reasonable bandwidth/latency
   and for avoiding the SDK's per-call overhead when the app comes back online after
   being offline for a while (e.g. 500 queued changes after a day of route editing).
3. Rows are marked `synced = 1` only after a 2xx response — safe to retry, safe to crash.
4. Synced rows older than 7 days are purged from `_sync_log` to keep the local DB small.
5. **`CLEAR_TABLE` is never fanned out client-side** into per-document deletes (that would
   require a read-all-then-delete-all, burning both read and delete quota). Instead it's
   deferred to a server-side Cloud Function trigple (recommended, optional) — or, if no
   Cloud Functions are used, simply skipped from sync (local-only reset), since a full
   table wipe is a rare admin operation, not a normal user action.

## Read path (pull): delta-only, on-demand, paginated — never a live listener

- **No `onSnapshot()` anywhere.** A live listener keeps a socket open and bills a read
  for every changed document across every connected client — the single fastest way to
  blow through 50K reads/day with more than a couple of users.
- Pulls happen only:
  - once at app launch,
  - on an explicit "Sync Now" button,
  - once, debounced, a few seconds after a local write burst settles (so a second device
    sees changes without polling).
- Each pull (`pullSync.ts::pullDeltasForTable`) queries `WHERE updatedAt > lastSyncedAt`,
  paginated at 300 docs/page, walking tables **sequentially** rather than firing 10
  parallel queries — this smooths the read burst and makes the daily budget easy to reason
  about: with ~10 sync tables and a few hundred changed rows/day, a realistic day uses
  low thousands of reads, comfortably under 50K even with several users polling.

## Storage

- All blobs (Excel exports, mascot image, etc.) stay local or are generated on-demand —
  nothing is uploaded to Firebase Storage, keeping the 1GiB/day download quota untouched.

## Auth

- Anonymous or email/password sign-in only (`firebase/client.ts`) — no paid Identity
  Platform features required.

## Budget summary (typical single-warehouse deployment, ~5 concurrent users)

| Operation | Frequency | Approx. Firestore ops/day |
|---|---|---|
| App launch delta pull (10 tables) | 5 users × ~3 launches | ~150–500 reads |
| Manual "Sync Now" | a few/user/day | ~200–600 reads |
| Outbox flush (batched commit) | every 45s while active, only non-empty batches send | a few hundred writes |
| Live listeners | **0** | 0 |

This sits at roughly 1–3% of the daily Spark quota under normal use, leaving headroom
for growth before any billing plan is needed.
