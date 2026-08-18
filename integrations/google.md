# Google

Status: planned. Calendar is the first integration (MVP milestone 1). Gmail is Phase two. Nothing is implemented in Stage 1.

## Google Calendar

Data: calendars (as sources) and events.

| Google | Scalar |
| --- | --- |
| calendar in `calendarList` | source |
| event | event (`source = google`, `sourceObjectId = event.id`, `sourceUrl = htmlLink`) |
| recurring event | expanded instances inside the sync window |
| cancelled event | deleted (marked, then removed on reconciliation) |

Sync: Google Calendar supports incremental sync with `syncToken` per calendar. The adapter stores the token as `sync_cursor`. A `410 Gone` means the token expired; the adapter drops it and performs a full listing. This matches [../architecture/sync.md](../architecture/sync.md).

Scopes: `https://www.googleapis.com/auth/calendar.readonly` for the read path. The write scope (`calendar.events`) is requested only when the user turns on scheduling proposals, and every write goes through the approval flow in [../architecture/ai-safety.md](../architecture/ai-safety.md).

## Gmail

Data: messages into the Inbox, with the ability to turn a message into a task.

Scopes: `gmail.readonly` initially. Sending on the user's behalf is a separate scope and a separate approval; it is Phase three at the earliest.

Sync: Gmail `history.list` with a stored `historyId` as the cursor, falling back to a full listing when the history id is too old.

## OAuth

Google integrations use OAuth 2.0 with refresh tokens (`access_type=offline`, `prompt=consent` on first connect). Token storage, rotation and revocation are described in [../security/oauth-and-tokens.md](../security/oauth-and-tokens.md).

Connecting Google as an integration is not the same as signing in with Google. See [../architecture/authorization.md](../architecture/authorization.md).

## Verification and quotas

A hosted deployment that requests restricted scopes (Gmail) needs Google's app verification. Self-hosters use their own Google Cloud project and OAuth client, configured through environment variables in `api` and `worker`.
