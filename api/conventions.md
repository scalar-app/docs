# API conventions

## Base path and format

- Base path `/api/v1`. `/health` is unversioned.
- Request and response bodies are JSON. Send `Content-Type: application/json`.
- Field names are camelCase.

## Identifiers

IDs are string UUIDs. The intent is UUIDv7-style ordering; Stage 1 uses `crypto.randomUUID()` (v4), which is acceptable for now. Do not parse or sort by id.

## Timestamps

ISO 8601 strings in UTC, for example `2026-08-18T14:00:00.000Z`. Date-only inputs (`/today?date=`) use `YYYY-MM-DD` together with an IANA `tz`.

## Errors

Every error response has this shape:

```json
{ "error": { "code": "SNAKE_CASE", "message": "Human sentence." } }
```

| HTTP | Code | When |
| --- | --- | --- |
| 400 | `INVALID_JSON` | Body is not valid JSON |
| 401 | `UNAUTHORIZED` | No or invalid session, or an invalid magic link |
| 404 | `NOT_FOUND` | Route unknown, or resource not in a workspace the user belongs to |
| 413 | `PAYLOAD_TOO_LARGE` | Body over 1 MB |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | Content-Type is not `application/json` |
| 422 | `VALIDATION_ERROR` | Body, query or params failed schema validation |
| 422 | `INVALID_CURSOR`, `INVALID_TIME_ZONE`, `INVALID_DATE`, `SPACE_NOT_IN_WORKSPACE`, `PARENT_TASK_NOT_IN_WORKSPACE`, `INVALID_PARENT_TASK` | Specific validation failures |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Unhandled failure; find the `x-request-id` in the server log. Stack traces are never returned |

Foreign resources return 404, not 403, so their existence is not revealed.

## Pagination

List endpoints are cursor based ([ADR 0008](../adr/0008-cursor-pagination.md)):

- Query: `?limit=&cursor=`. `limit` is 1 to 200, default 50.
- Response: `{ "data": [...], "nextCursor": string | null }`.
- `nextCursor` is opaque. Pass it back unchanged. `null` means no more pages.

## Headers

- Every response carries `x-request-id`. If the client sends one it is echoed, otherwise the server generates it. Include it in bug reports.

## Cookies and sessions

- Session cookie name: `scalar_session`.
- Attributes: `HttpOnly`, `SameSite=Lax`, `Secure` in production, `Path=/`.
- The cookie holds an opaque random token, signed with `COOKIE_SECRET`. Only its SHA-256 hash is stored in the `sessions` table.
- Browser clients must send requests with credentials included. There is no bearer token in Stage 1.

## Workspace scoping

All resources are scoped by workspace membership. Stage 1 users have exactly one personal workspace, so endpoints do not take a workspace parameter yet; the server resolves it from the session. When multiple workspaces exist, a workspace selector will be added without changing response shapes.

## Filters

Query filters are documented per endpoint. Comma lists (`status=todo,in_progress`) are used for enums. Unknown query parameters are ignored.
