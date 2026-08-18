# Events

Read-only in Stage 1. The `events` table exists; nothing populates it until calendar sync is implemented. Native event creation is planned.

## Event shape

```json
{
  "id": "…",
  "workspaceId": "…",
  "title": "Lecture",
  "description": null,
  "startsAt": "2026-08-19T09:00:00.000Z",
  "endsAt": "2026-08-19T10:00:00.000Z",
  "allDay": false,
  "location": null,
  "source": "google",
  "sourceObjectId": "abc123",
  "sourceUrl": "https://calendar.google.com/…",
  "createdAt": "…",
  "updatedAt": "…"
}
```

`source` is a provider string: `google`, `canvas`, or `scalar` for events created natively.

## GET /api/v1/events?from=&to=

| Param | Required | Meaning |
| --- | --- | --- |
| `from` | yes | ISO timestamp, inclusive lower bound on `startsAt` (events overlapping the range are included) |
| `to` | yes | ISO timestamp, exclusive upper bound |
| `limit`, `cursor` | no | See [conventions](../conventions.md) |

Response: `{ "data": Event[], "nextCursor": string | null }`. Events are ordered by `startsAt` then `id`; the cursor pages on that order.

`422 VALIDATION_ERROR` if `from` or `to` is missing or `to <= from`.

## Not in Stage 1

- `POST`, `PATCH`, `DELETE` for events.
- Recurrence. Synced recurring events will be stored as expanded instances within the sync window.
- Attendees, reminders.
