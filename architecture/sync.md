# Sync

Status: implemented for Google Calendar. The worker runs syncs, the API owns the database and the cursors. Other providers are planned and will follow the same shape.

## Principles

- Cursor based and incremental. Each provider exposes some form of change token (Google `syncToken`, Canvas `updated_since` style filters or full listing with ETag). Scalar stores the cursor per account and per source and asks only for what changed.
- Idempotent. Imported events are upserted on `(integration_account_id, source_object_id)`. Running the same sync twice produces the same rows.
- Retries with exponential backoff and jitter. Transient failures (5xx, network) are retried; 4xx other than 429 are not.
- Rate limits respected. 429 and provider quota headers set `next_sync_at` in the future instead of retrying immediately.
- Deduplication. The same real-world object reachable through two sources (an event on two calendars, an assignment also present as a calendar entry) is linked, not duplicated, using provider ids first and heuristics second.
- Reconciliation. A periodic full listing catches deletions and missed changes that incremental cursors cannot report. Objects absent from the full listing are marked deleted, not hard-deleted.
- Provenance kept. See the provenance columns in [data-model.md](data-model.md).

## Per-account state

`integration_sync_state` holds one row per synced resource (for Google Calendar, per calendar):

| Field | Meaning |
| --- | --- |
| `last_successful_sync` | Timestamp of the last run that finished without error |
| `sync_cursor` | Opaque provider cursor or token |
| `sync_status` | `idle`, `queued`, `running`, `error` |
| `last_error` | Message and code of the last failure, cleared on success |
| `next_sync_at` | Earliest time the next run may start (drives backoff and rate limiting) |

## Flow

```text
api                         queue (Redis)              worker
---                         -------------              ------
POST /integrations/...  ->  enqueue sync job    ->     claim job
  connect / manual sync                                GET  /internal/v1/.../context
                                                         (access token, cursor per resource)
                                                       call provider with the cursor
                                            <-         POST /internal/v1/.../result
apply result in one transaction                          (upserts, deletes, next cursor)
  upsert events, delete cancelled
  store cursor, last_successful_sync
  set next_sync_at
                                            <-         POST /internal/v1/sync/schedule
enqueue every due resource                               (every few minutes)
```

The API never calls a provider inline: it records intent (a job) and returns. The worker holds no database connection and no credentials of its own; it asks the API for a short lived access token, and the API refreshes that token when needed. On failure the worker reports the error, the API records it, and the queue retries with exponential backoff. A revoked grant is terminal: the account moves to `reauthorization_required` and the user is asked to reconnect.

## Failure handling

Jobs get five attempts with exponential backoff starting at 30 seconds. The resource's `sync_status` becomes `error` with `last_error` set, which the integration settings page shows. Failed jobs are kept in Redis for seven days for inspection; completed jobs are removed immediately so a manual sync is never deduplicated against a finished run. Manual retry from the interface enqueues a fresh job.

## Decisions taken

- `worker` and `integrations` are separate repositories: `integrations` is pure provider code with no infrastructure, so the API can use it for OAuth while the worker uses it for syncing.
- BullMQ over Redis, which is already part of the stack. Queue name `scalar.sync`, job name `integration.sync`, job ids derived from account and resource so a pending job is not duplicated.
