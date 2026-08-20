# API

The Scalar HTTP API is served by the `api` repository (Fastify, TypeScript). Base path: `/api/v1`. The health check is at `/health` (no version prefix).

The `@scalar/sdk` package mirrors this contract; see [../sdk/README.md](../sdk/README.md).

## Pages

- [conventions.md](conventions.md): JSON shapes, errors, pagination, timestamps, ids, headers, cookies
- [v1/auth.md](v1/auth.md): magic link login, sessions, `me`, workspaces
- [v1/tasks.md](v1/tasks.md)
- [v1/spaces.md](v1/spaces.md)
- [v1/projects.md](v1/projects.md)
- [v1/preferences.md](v1/preferences.md)
- [v1/events.md](v1/events.md)
- [v1/home.md](v1/home.md)
- [v1/inbox.md](v1/inbox.md)
- [v1/today.md](v1/today.md)
- [v1/timeline.md](v1/timeline.md)
- [v1/planner.md](v1/planner.md)
- [v1/focus.md](v1/focus.md)
- [v1/integrations.md](v1/integrations.md)
- [v1/search.md](v1/search.md)
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
| GET, POST | `/api/v1/projects` |
| GET, PATCH, DELETE | `/api/v1/projects/:id` |
| GET, PATCH | `/api/v1/preferences` |
| GET, POST | `/api/v1/tasks` |
| GET, PATCH, DELETE | `/api/v1/tasks/:id` |
| GET | `/api/v1/events?from=&to=` |
| GET | `/api/v1/home?date=&tz=` |
| GET | `/api/v1/diagnostics` |
| GET | `/api/v1/inbox` |
| POST | `/api/v1/inbox/:taskId/accept` |
| POST | `/api/v1/inbox/:taskId/dismiss` |
| GET | `/api/v1/today?date=&tz=` |
| GET | `/api/v1/timeline?date=&tz=` |
| POST | `/api/v1/planner/preview` |
| POST | `/api/v1/planner/apply` |
| GET | `/api/v1/focus/current` |
| POST | `/api/v1/focus/start` |
| POST | `/api/v1/focus/:id/complete` |
| POST | `/api/v1/focus/:id/cancel` |
| GET | `/api/v1/focus/sessions` |
| GET | `/api/v1/search?q=&limit=` |
| GET | `/api/v1/integrations` |
| POST | `/api/v1/integrations/google/connect` |
| POST | `/api/v1/integrations/canvas/connect` |
| POST | `/api/v1/integrations/:id/sync` |
| DELETE | `/api/v1/integrations/:id?data=keep\|delete` |
| POST | `/api/v1/command` |
| GET | `/api/v1/command/threads` |
| GET | `/api/v1/command/threads/:id` |
| POST | `/api/v1/command/actions/:id/approve` |
| POST | `/api/v1/command/actions/:id/reject` |

Everything except `/health`, `POST /auth/magic-link` and `GET /auth/magic-link/verify` requires a session.

## Not implemented

- Any write endpoint for events. Calendar entries arrive through sync.
- Notifications.

Command endpoints require a model provider on the server (`AI_PROVIDER`, see [deployment/ai-providers.md](../deployment/ai-providers.md)). Without one they return `503 AI_UNAVAILABLE` and everything else works normally. `GET /api/v1/command/status` reports which provider is live, and never fails when there is none.
