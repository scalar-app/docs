# Google

Status: Google Calendar is implemented (read only). Gmail is planned.

## Google Calendar

Data: calendars (as sources) and events.

| Google | Scalar |
| --- | --- |
| calendar in `calendarList` | source |
| event | event (`source = google_calendar`, `sourceObjectId = event.id`, `sourceUrl = htmlLink`) |
| recurring event | expanded instances inside the sync window |
| cancelled event | deleted from `events` on the next sync |

Sync: incremental with a `syncToken` per calendar, stored as `sync_cursor`. The first run is a windowed full sync (30 days back by default) with `singleEvents=true` and `showDeleted=true`. A `410 Gone` means the token expired; the provider code performs a full resync transparently and reports `fullResync`. See [../architecture/sync.md](../architecture/sync.md).

Calendars: on connect, Scalar lists `calendarList` and syncs the primary plus any calendar the user has selected in Google. One `integration_sync_state` row exists per calendar.

All day events carry a `date` rather than a `dateTime`; they are stored at UTC midnight with `allDay = true` so the day does not shift with the viewer's zone.

Scopes: `calendar.readonly` and `userinfo.email` (the latter so the interface can show which account is connected). The write scope (`calendar.events`) is requested only when the user turns on scheduling proposals, and every write goes through the approval flow in [../architecture/ai-safety.md](../architecture/ai-safety.md).

## Gmail

Data: messages into the Inbox, with the ability to turn a message into a task.

Scopes: `gmail.readonly` initially. Sending on the user's behalf is a separate scope and a separate approval; it is Phase three at the earliest.

Sync: Gmail `history.list` with a stored `historyId` as the cursor, falling back to a full listing when the history id is too old.

## OAuth

Google integrations use OAuth 2.0 with refresh tokens (`access_type=offline`, `prompt=consent` on first connect). Token storage, rotation and revocation are described in [../security/oauth-and-tokens.md](../security/oauth-and-tokens.md).

Connecting Google as an integration is not the same as signing in with Google. See [../architecture/authorization.md](../architecture/authorization.md).

## Verification and quotas

A hosted deployment that requests restricted scopes (Gmail) needs Google's app verification. Self-hosters use their own Google Cloud project and OAuth client, configured through `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` in both `api` and `worker`. The authorized redirect URI is `${API_ORIGIN}/api/v1/integrations/google/callback` and must match exactly.

## Endpoints

`POST /api/v1/integrations/google/connect` returns the consent URL. `GET /api/v1/integrations/google/callback` consumes the single use state, exchanges the code, stores encrypted tokens, discovers calendars, queues the first sync and redirects to `${APP_ORIGIN}/settings/integrations`. `POST /api/v1/integrations/:id/sync` queues a run. `DELETE /api/v1/integrations/:id?data=keep|delete` revokes and disconnects.
