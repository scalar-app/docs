# Architecture decision records

Decisions that shape Scalar and are expensive to reverse. Each record states the context at the time, the decision, and the consequences we accepted. Records are immutable once accepted; a later record supersedes an earlier one rather than editing it.

| Number | Title | Status |
| --- | --- | --- |
| [0001](0001-postgresql-primary-datastore.md) | PostgreSQL as the primary datastore | Accepted |
| [0002](0002-tauri-for-desktop.md) | Tauri for the desktop application | Accepted |
| [0003](0003-repo-per-concern-not-monorepo.md) | One repository per concern, not a monorepo | Accepted |
| [0004](0004-fastify-and-drizzle-for-api.md) | Fastify and Drizzle ORM for the API | Accepted |
| [0005](0005-agpl-3-license.md) | AGPL-3.0 license | Accepted |
| [0006](0006-nextjs-for-web-app.md) | Next.js for the web application | Accepted |
| [0007](0007-astro-for-public-website.md) | Astro for the public website | Accepted |
| [0008](0008-cursor-pagination.md) | Cursor based pagination | Accepted |

## Template

```markdown
# NNNN. Title

Date: YYYY-MM-DD
Status: Proposed | Accepted | Superseded by NNNN

## Context
## Decision
## Consequences
## Alternatives considered
```

Add a record when a decision constrains more than one repository, introduces or replaces a core dependency, or would surprise a new contributor without the reasoning.
