# Today

`GET /api/v1/today?date=YYYY-MM-DD&tz=` returns a deterministic view of one day. No AI is involved; the same inputs always give the same output.

## Query

| Param | Required | Meaning |
| --- | --- | --- |
| `date` | no (defaults to today in `tz`) | `YYYY-MM-DD` |
| `tz` | no (defaults to `UTC`) | IANA time zone, for example `Europe/Berlin` |

The day boundaries are computed in `tz` and converted to UTC for the queries.

## Response

```json
{
  "date": "2026-08-18",
  "greeting": "Good afternoon.",
  "attentionCount": 3,
  "urgent": [Task],
  "upcoming": [Event],
  "dueToday": [Task],
  "overdue": [Task]
}
```

| Field | Definition |
| --- | --- |
| `date` | The requested date |
| `greeting` | Based on the current hour in `tz`: `Good morning.` (5 to 11), `Good afternoon.` (12 to 16), `Good evening.` otherwise |
| `urgent` | Open tasks with `priority` `high` or `urgent` and `dueAt` within the next 48 hours |
| `dueToday` | Open tasks with `dueAt` inside the day |
| `overdue` | Open tasks with `dueAt` before the start of the day |
| `upcoming` | Events overlapping the day, ordered by `startsAt` |
| `attentionCount` | Number of distinct tasks across `urgent`, `dueToday`, `overdue` |

"Open" means status not in `done`, `cancelled`. A task can appear in more than one list (for example both `urgent` and `dueToday`); `attentionCount` counts it once.

Errors: `422 INVALID_TIME_ZONE` for an unknown zone, `422 INVALID_DATE` for a malformed or impossible date.

The rules live in `src/modules/today/compute.ts` in the `api` repository as a pure function with unit tests. If this page and the code disagree, the code wins; fix the page.
