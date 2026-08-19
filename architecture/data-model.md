# Data model

PostgreSQL is the primary datastore ([ADR 0001](../adr/0001-postgresql-primary-datastore.md)). The schema is owned by the `api` repository and managed with Drizzle migrations ([ADR 0004](../adr/0004-fastify-and-drizzle-for-api.md)).

IDs are UUID strings. Timestamps are stored as `timestamptz` and serialized as ISO 8601 UTC. Every workspace-scoped table carries `workspace_id`.

## Implementation status

Stage 1 defines these tables in `api`:

`users`, `sessions`, `magic_link_tokens`, `workspaces`, `workspace_members`, `spaces`, `tasks`, `events`.

The V2 foundation migration (`0003_v2_foundation`) adds `projects` and `user_preferences`, and gives `tasks` a `project_id` plus the same provenance columns `events` already had. It is additive: no table is dropped and no column is repurposed.

Everything else below is planned and does not exist in a migration yet. Column lists are a design target; the Drizzle schema in `api` is the source of truth for exact column names and types.

## Core tables

### Identity and tenancy

| Table | Status | Purpose |
| --- | --- | --- |
| `users` | implemented | `id`, `email` (unique), `name`, `created_at`, `updated_at` |
| `sessions` | implemented | Server-side session backing the `scalar_session` cookie: `id`, `user_id`, `expires_at`, `created_at` |
| `magic_link_tokens` | implemented | Hashed one-time tokens: `id`, `email`, `token_hash`, `expires_at`, `used_at`, `created_at` |
| `accounts` | planned | Login providers other than magic link (Google sign-in etc.) |
| `workspaces` | implemented | `id`, `name`, `type` (`personal` for now), `created_at`, `updated_at`. Each user gets a personal workspace on first login |
| `workspace_members` | implemented | `workspace_id`, `user_id`, `role` (`owner` for now), `created_at` |

### Organization

| Table | Status | Purpose |
| --- | --- | --- |
| `spaces` | implemented | `id`, `workspace_id`, `name`, `description`, `color`, `icon`, `created_by`, `archived_at`, `created_at`, `updated_at`. A course, project or area |
| `projects` | implemented | `id`, `workspace_id`, `space_id`, `name`, `description`, `status` (`active`, `paused`, `completed`, `archived`), `start_at`, `due_at`, `created_by`, plus provenance (`source`, `integration_account_id`, `source_object_id`, `source_url`, `last_synced_at`) unique on `(integration_account_id, source_object_id)`. A body of work inside a Space, or the mirror of something on a provider such as a Canvas course. Deleting a Space clears `space_id` rather than cascading |
| `tasks` | implemented | See task shape below. Carries `project_id` and provenance (`source`, `integration_account_id`, `source_object_id`, `source_url`, `source_updated_at`, `last_synced_at`), unique on `(integration_account_id, source_object_id)` so repeated syncs are idempotent. `source_id` is the superseded V1 column, kept until a later release drops it |
| `user_preferences` | implemented | One row per person: `time_zone`, `week_starts_on`, `workday_start_minute`, `workday_end_minute`, `work_days`, `default_focus_minutes`, `minimum_buffer_minutes`, `auto_schedule`, `duration_learning_enabled`. The planner's inputs. Personal rather than workspace scoped |
| `task_dependencies` | planned | `task_id`, `depends_on_task_id` |
| `events` | implemented | `id`, `workspace_id`, `title`, `description`, `starts_at`, `ends_at`, `all_day`, `location`, `source`, `integration_account_id`, `source_object_id`, `source_url`, `source_updated_at`, `last_synced_at`, `created_at`, `updated_at`. Written by integration sync; unique on `(integration_account_id, source_object_id)` so upserts are idempotent |
| `focus_sessions` | implemented | `id`, `workspace_id`, `user_id`, `task_id`, `status`, `planned_minutes`, `started_at`, `ended_at`, `actual_minutes`, `notes`. Scoped to the person, not the workspace. A partial unique index on `user_id where status = 'active'` makes "one thing at a time" a database rule rather than a convention |
| `timeline_blocks` | planned | Blocks that exist in their own right (focus, personal, buffer). The day's sequence is a read model over `events` and scheduled `tasks` today, so nothing is stored twice |
| `task_suggestions` | implemented | A proposed patch to an inbox item: `workspace_id`, `task_id`, `status` (`pending`, `accepted`, `edited`, `dismissed`), `origin`, `source`, `suggestion` (JSON), `reason`, `decided_at`. Advisory only; nothing is applied until a person accepts it. There is no `inbox_items` table: an unfiled task is a task with `status = 'inbox'` |
| `notifications` | planned | In-app notifications |

### Integrations and provenance

| Table | Status | Purpose |
| --- | --- | --- |
| `integration_accounts` | implemented | One row per connected external account: `workspace_id`, `user_id`, `provider`, `external_account_id`, `display_name`, `status` (`active`, `reauthorization_required`, `disconnected`), `settings`, `connected_at`, `disconnected_at`. Unique on `(workspace_id, provider, external_account_id)` |
| `integration_tokens` | implemented | One row per account: AES-256-GCM ciphertext for the access and refresh tokens, expiry, scopes. Read only by the credential service. See [security/oauth-and-tokens.md](../security/oauth-and-tokens.md) |
| `integration_sync_state` | implemented | One row per synced resource (a calendar): `sync_cursor`, `sync_status`, `last_successful_sync_at`, `last_attempt_at`, `last_error`, `next_sync_at`. See [sync.md](sync.md) |
| `sources` | planned | Logical source inside an account, for example one Google calendar or one Canvas course |
| `source_objects` | planned | Raw imported objects keyed by provider id, kept for reconciliation and re-derivation |

### AI

| Table | Status | Purpose |
| --- | --- | --- |
| `ai_threads` | planned | Conversation containers per workspace |
| `ai_messages` | planned | Messages with role and content |
| `ai_actions` | planned | Proposed tool calls with classification, approval state, result. See [ai-safety.md](ai-safety.md) |

### Audit

| Table | Status | Purpose |
| --- | --- | --- |
| `audit_logs` | planned | Who did what to which resource in which workspace |

## Provenance columns

Any table that can hold imported data carries the same set of columns so that origin is never lost:

| Column | Meaning |
| --- | --- |
| `source_provider` | `google`, `canvas`, `scalar` for native |
| `source_account_id` | FK to `integration_accounts` |
| `source_object_id` | Provider's own identifier |
| `source_url` | Deep link to the object in the provider |
| `source_created_at` | Creation time reported by the provider |
| `source_updated_at` | Last modification reported by the provider |
| `last_synced_at` | When Scalar last reconciled this row |

Stage 1 `events` uses a shorter form (`source`, `source_object_id`, `source_url`) that will be aligned with this list when integrations arrive. `tasks` has `source_id` reserved for the same purpose.

## Task

Task status enum:

```text
inbox | todo | in_progress | blocked | done | cancelled
```

Priority enum:

```text
none | low | medium | high | urgent
```

Task columns as exposed by the API (camelCase in JSON, snake_case in the table):

`id`, `workspaceId`, `spaceId | null`, `title`, `description | null`, `status`, `priority`, `dueAt | null`, `scheduledStart | null`, `scheduledEnd | null`, `estimatedMinutes | null`, `sourceId | null`, `parentTaskId | null`, `createdBy`, `createdAt`, `updatedAt`, `completedAt | null`.

`completedAt` is set when status becomes `done` and cleared when it leaves `done`.

## Indexes

Expected indexes; check the migrations in `api` for what actually exists:

- `tasks (workspace_id, status)` and `tasks (workspace_id, due_at)` for Today and filters.
- `events (workspace_id, starts_at)` for range queries.
- `magic_link_tokens (token_hash)` unique.
- Planned: unique `(source_provider, source_account_id, source_object_id)` on imported tables for idempotent upserts.
