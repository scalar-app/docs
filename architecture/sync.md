# Sync

Status: planned. No sync code exists in Stage 1. The `events` table exists so the read path can be built first; nothing writes to it yet.

## Principles

- Cursor based and incremental. Each provider exposes some form of change token (Google `syncToken`, Canvas `updated_since` style filters or full listing with ETag). Scalar stores the cursor per account and per source and asks only for what changed.
- Idempotent. Every imported object is upserted on `(source_provider, source_account_id, source_object_id)`. Running the same sync twice produces the same rows.
- Retries with exponential backoff and jitter. Transient failures (5xx, network) are retried; 4xx other than 429 are not.
- Rate limits respected. 429 and provider quota headers set `next_sync_at` in the future instead of retrying immediately.
- Deduplication. The same real-world object reachable through two sources (an event on two calendars, an assignment also present as a calendar entry) is linked, not duplicated, using provider ids first and heuristics second.
- Reconciliation. A periodic full listing catches deletions and missed changes that incremental cursors cannot report. Objects absent from the full listing are marked deleted, not hard-deleted.
- Provenance kept. See the provenance columns in [data-model.md](data-model.md).

## Per-account state

`integration_sync_state` (planned) holds one row per integration account, and optionally per source:

| Field | Meaning |
| --- | --- |
| `last_successful_sync` | Timestamp of the last run that finished without error |
| `sync_cursor` | Opaque provider cursor or token |
| `sync_status` | `idle`, `running`, `error`, `disabled` |
| `last_error` | Message and code of the last failure, cleared on success |
| `next_sync_at` | Earliest time the next run may start (drives backoff and rate limiting) |

## Flow

```text
api                         queue (Redis)              worker
---                         -------------              ------
POST /integrations/...  ->  enqueue sync job    ->     claim job
  connect / manual sync                                load sync state
                                                       call provider with cursor
                                                       upsert source_objects
                                                       derive tasks / events / inbox_items
                                                       store new cursor, last_successful_sync
                                                       schedule next run (next_sync_at)
                                            <-         on failure: backoff, last_error, retry
                                                       after N failures: dead-letter
```

The API never calls a provider inline. It records intent (a job) and returns. Scheduled syncs are enqueued by the worker itself based on `next_sync_at`.

## Dead-letter

Jobs that exhaust their retries move to a dead-letter queue. The account's `sync_status` becomes `error` with `last_error` set. The user sees this on the integration settings page and can retry manually, which re-enqueues the job and resets the retry counter. Dead-letter contents are kept for inspection and are not silently dropped.

## Open questions

- Whether `worker` and `integrations` are one repository or two. Currently leaning towards one until the adapter count grows.
- Queue library choice (BullMQ is the default candidate because Redis is already part of the stack).
