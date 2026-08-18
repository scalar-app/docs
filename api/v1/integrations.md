# Integrations

Connected external accounts and their sync state. Implemented for Google Calendar; see [../../integrations/google.md](../../integrations/google.md) for provider specifics.

All routes require a session and operate on the caller's workspace.

## Account shape

```json
{
  "id": "…",
  "provider": "google_calendar",
  "displayName": "you@example.com",
  "status": "active",
  "connectedAt": "2026-08-18T19:11:01.809Z",
  "resources": [
    {
      "resourceId": "primary",
      "resourceName": "Personal",
      "syncStatus": "idle",
      "lastSuccessfulSyncAt": "2026-08-18T19:11:01.881Z",
      "lastAttemptAt": "2026-08-18T19:11:01.881Z",
      "lastError": null,
      "nextSyncAt": "2026-08-18T19:26:01.881Z"
    }
  ]
}
```

`status`: `active`, `reauthorization_required` (the provider revoked access; the user must reconnect), `disconnected` (never returned in listings).

`syncStatus`: `idle`, `queued`, `running`, `error`.

Credentials are never part of any response.

## GET /api/v1/integrations

Lists connected accounts, newest first. Response `{ "data": IntegrationAccount[] }`. Disconnected accounts are omitted.

## POST /api/v1/integrations/google/connect

Returns `{ "url": "https://accounts.google.com/o/oauth2/v2/auth?..." }`. Send the browser there. The URL carries a random single use `state` bound to the requesting user and stored in Redis for 10 minutes.

`503 PROVIDER_NOT_CONFIGURED` when the server has no Google OAuth client.

## GET /api/v1/integrations/google/callback

Google redirects here. Scalar consumes the state, exchanges the code, stores encrypted tokens, creates or reactivates the account, discovers calendars, queues the first sync, and redirects the browser to `${APP_ORIGIN}/settings/integrations` with query parameters:

| `result` | Meaning |
| --- | --- |
| `connected` | Success. `account` carries the account id |
| `denied` | The user declined consent |
| `error` | Something failed. `code` carries the error code, for example `OAUTH_STATE_INVALID` |

This endpoint always redirects; it never returns JSON.

## POST /api/v1/integrations/:id/sync

Queues a sync for every resource on the account. Response `202 { "ok": true }`. A job already waiting is not duplicated.

`409 INTEGRATION_NOT_ACTIVE` when the account needs reauthorization. `404 NOT_FOUND` for another workspace's account.

## DELETE /api/v1/integrations/:id

Disconnects. Query `data`:

| Value | Effect on imported events |
| --- | --- |
| `keep` (default) | Kept, with the link to the account removed |
| `delete` | Deleted |

In both cases Scalar revokes the token at the provider (best effort), deletes stored credentials, and stops syncing. Response `200 { "ok": true }`.

## Internal endpoints

`/internal/v1/sync/*` serves the worker and is authenticated with a shared secret, not a session. It is not part of the public contract and is not in the SDK. See [../../architecture/sync.md](../../architecture/sync.md).
