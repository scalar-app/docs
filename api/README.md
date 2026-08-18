# API

The Scalar HTTP API is served by the `api` repository (Fastify, TypeScript). Base path: `/api/v1`. The health check is at `/health` (no version prefix).

The `@scalar/sdk` package mirrors this contract; see [../sdk/README.md](../sdk/README.md).

## Pages

- [conventions.md](conventions.md): JSON shapes, errors, pagination, timestamps, ids, headers, cookies
- [v1/auth.md](v1/auth.md): magic link login, sessions, `me`, workspaces
- [v1/tasks.md](v1/tasks.md)
- [v1/spaces.md](v1/spaces.md)
- [v1/events.md](v1/events.md)
- [v1/today.md](v1/today.md)
- [v1/integrations.md](v1/integrations.md)
- [v1/command.md](v1/command.md)

## Endpoint list

| Method | Path |
| --- | --- |
| GET | `/health` |
| POST | `/api/v1/auth/magic-link` |
| GET | `/api/v1/auth/magic-link/verify?token=` |
| POST | `/api/v1/auth/logout` |
| GET | `/api/v1/me` |
| GET | `/api/v1/workspaces` |
| GET, POST | `/api/v1/spaces` |
| GET, PATCH, DELETE | `/api/v1/spaces/:id` |
| GET, POST | `/api/v1/tasks` |
| GET, PATCH, DELETE | `/api/v1/tasks/:id` |
| GET | `/api/v1/events?from=&to=` |
| GET | `/api/v1/today?date=&tz=` |
| GET | `/api/v1/integrations` |
| POST | `/api/v1/integrations/google/connect` |
| POST | `/api/v1/integrations/:id/sync` |
| DELETE | `/api/v1/integrations/:id?data=keep\|delete` |
| POST | `/api/v1/command` |
| GET | `/api/v1/command/threads` |
| GET | `/api/v1/command/threads/:id` |
| POST | `/api/v1/command/actions/:id/approve` |
| POST | `/api/v1/command/actions/:id/reject` |

Everything except `/health`, `POST /auth/magic-link` and `GET /auth/magic-link/verify` requires a session.

## Not implemented

- Email delivery for magic links (dev logs the link).
- Any write endpoint for events. Calendar entries arrive through sync.
- Inbox, notifications, search.

Command endpoints require `ANTHROPIC_API_KEY` on the server. Without it they return `503 AI_UNAVAILABLE` and everything else works normally.
