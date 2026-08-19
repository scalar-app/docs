# Auth, me, workspaces

Authentication is by magic link and server side sessions. Tokens are stored hashed, sessions are rows in the database, and the cookie is `HttpOnly`. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/auth`).

## POST /api/v1/auth/magic-link

Request a sign in link.

Request:

```json
{ "email": "user@example.com" }
```

The email is trimmed and lower cased.

Response `200`:

```json
{ "ok": true, "devLink": "http://localhost:3000/auth/verify?token=..." }
```

`devLink` is present only when the API runs outside production (`NODE_ENV` is not `production`). The response is the same whether or not the email is known.

Behaviour:

- Creates a `magic_link_tokens` row holding the SHA-256 hash of a random token. The plain token is never stored.
- The link points at `${APP_ORIGIN}/auth/verify?token=...` and expires after `MAGIC_LINK_TTL_MINUTES` (default 15).
- Rate limited in Redis: 5 requests per email and 20 per IP in a 15 minute window. Over the limit returns `429 RATE_LIMITED`.
- Email delivery uses SMTP when configured. Outside production the link is logged and returned as `devLink`. In production the request succeeds, nothing is sent, and a warning is logged.

## GET /api/v1/auth/magic-link/verify?token=

Consumes the token (single use), creates the user and a personal workspace on first sign in (inside one transaction), creates a session and sets the `scalar_session` cookie (`HttpOnly`, `SameSite=Lax`, `Path=/`, signed, `Secure` in production, expires after `SESSION_TTL_DAYS`, default 30).

Response `200`:

```json
{
  "user": { "id": "…", "email": "user@example.com", "name": null, "createdAt": "…", "updatedAt": "…" },
  "workspace": { "id": "…", "name": "user's workspace", "ownerId": "…", "kind": "personal", "role": "owner", "createdAt": "…", "updatedAt": "…" }
}
```

Errors: `422 VALIDATION_ERROR` when the token is missing or malformed, `401 UNAUTHORIZED` when it is unknown, expired or already used.

## POST /api/v1/auth/logout

Deletes the session row and clears the cookie. Response `200 { "ok": true }`. Safe to call without a session.

## GET /api/v1/me

Requires a session. Returns the current user and their workspace:

```json
{ "user": User, "workspace": Workspace }
```

`401 UNAUTHORIZED` without a valid session.

## GET /api/v1/workspaces

Requires a session. Lists workspaces the user belongs to. Every user has at least one personal workspace.

```json
{ "data": [Workspace] }
```

This list is not paginated; a user has few workspaces.

## Shapes

`User`: `id`, `email`, `name` (nullable), `createdAt`, `updatedAt`.

`Workspace`: `id`, `name`, `ownerId`, `kind` (`personal` or `team`), `role` (`owner`, `admin` or `member`), `createdAt`, `updatedAt`.

## Not yet

Google, GitHub, Microsoft and Apple sign in; email delivery; multiple workspaces per user in the UI.
