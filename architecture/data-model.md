# Data model

PostgreSQL is the primary datastore ([ADR 0001](../adr/0001-postgresql-primary-datastore.md)). The schema is owned by the `api` repository and managed with Drizzle migrations ([ADR 0004](../adr/0004-fastify-and-drizzle-for-api.md)).

IDs are UUID strings. Timestamps are stored as `timestamptz` and serialized as ISO 8601 UTC. Every workspace-scoped table carries `workspace_id`.

## Implementation status

Stage 1 defines these tables in `api`:

`users`, `sessions`, `magic_link_tokens`, `workspaces`, `workspace_members`, `spaces`, `tasks`, `events`.

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
| `tasks` | implemented | See task shape below |
| `task_dependencies` | planned | `task_id`, `depends_on_task_id` |
| `events` | implemented (table only) | `id`, `workspace_id`, `title`, `description`, `starts_at`, `ends_at`, `all_day`, `location`, `source`, `source_object_id`, `source_url`, `created_at`, `updated_at`. Read-only in Stage 1; populated by integrations later |
| `inbox_items` | planned | Untriaged items (emails, announcements) with provenance |
| `notifications` | planned | In-app notifications |

### Integrations and provenance

| Table | Status | Purpose |
| --- | --- | --- |
| `integration_accounts` | planned | One row per connected external account: `provider`, `external_account_id`, `display_name`, `status` |
| `integration_tokens` | planned | Encrypted access and refresh tokens, expiry, scopes. See [security/oauth-and-tokens.md](../security/oauth-and-tokens.md) |
| `integration_sync_state` | planned | Per-account cursor and health. See [sync.md](sync.md) |
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
