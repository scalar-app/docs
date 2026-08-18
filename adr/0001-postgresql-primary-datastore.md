# 0001. PostgreSQL as the primary datastore

Date: 2026-08-18
Status: Accepted

## Context

Scalar stores users, workspaces, tasks, events, imported items with provenance, integration state and eventually audit logs. The data is relational (tasks belong to spaces and workspaces, items reference sources), needs transactions (creating a user and their workspace atomically, consuming a single use token), and needs full text search early. Self-hosters must be able to run it without a managed cloud service.

## Decision

PostgreSQL is the only primary datastore. Redis is used for ephemeral state (rate limits, queues, caches), S3-compatible storage for blobs. Full text search starts with PostgreSQL built ins; a dedicated search engine is introduced only if scale demands it.

## Consequences

- One system to back up, migrate and reason about. Schema changes go through migrations, never manual edits.
- Search quality is bounded by PostgreSQL until a search service exists; that is acceptable for Stage 1 volumes.
- Contributors need Docker or a local PostgreSQL. The `api` compose file provides it.

## Alternatives considered

- SQLite: simpler locally, but multi user hosting, concurrent sync workers and full text needs point away from it.
- Document stores: the data is relational and integrity matters more than schema flexibility.
- Managed proprietary databases: would tie self-hosting to one vendor.
