# Home

The answer to "what should I be doing right now", plus what needs a decision. One request, computed deterministically.

**No model is called to load Home.** The rules are ordinary code, they give the same answer every time, and everything surfaced carries the reason it was surfaced. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/home`), with the reasoning itself in `compute.ts` as pure functions.

## GET /api/v1/home

Query: `date` (YYYY-MM-DD, defaults to today in `tz`), `tz` (defaults to `UTC`; the SDK sends the reader's own zone).

```json
{
  "date": "2026-08-19",
  "greeting": "Good morning.",
  "timeZone": "America/Los_Angeles",
  "busyMinutes": 195,
  "upNext": {
    "kind": "scheduled_task",
    "itemId": "9f…",
    "taskId": "9f…",
    "title": "Finish problem set",
    "startAt": "2026-08-19T18:00:00.000Z",
    "endAt": "2026-08-19T18:45:00.000Z",
    "estimatedMinutes": null,
    "reason": "next_scheduled"
  },
  "attention": [
    {
      "id": "not_enough_time:2a…",
      "kind": "not_enough_time",
      "title": "CSE homework",
      "detail": "Needs 2 hr. There is 1 hr 20 min of free working time before it is due.",
      "taskId": "2a…"
    }
  ]
}
```

## Up Next

Checked in order, first match wins:

1. **A running focus session.** The brief puts a current calendar event first; Scalar puts the session first, because a session is a decision the person made a moment ago while an event is an inference from a calendar. When someone has said "this is what I am doing", Scalar should not argue.
2. **Whatever is happening now**: a block whose start has passed and whose end has not.
3. **The next scheduled task**, ahead of a later event: work someone planned is work they meant to do next.
4. **The next event.**
5. **The most pressing task with no time yet**, ordered by deadline then priority then id.
6. Otherwise `kind: "nothing"`.

All-day blocks never count as "happening now", or every day with a holiday on it would answer with the holiday.

## Needs attention

Ordered by how much it costs to ignore: overdue, not enough time, due soon, conflict, urgent and unscheduled, disconnected integration, failing sync.

| Kind | When |
| --- | --- |
| `overdue` | An open task's deadline has passed. Says how long ago |
| `not_enough_time` | The work needs more free working time than remains before its deadline |
| `due_soon` | Due within 24 hours, with enough time to do it |
| `schedule_conflict` | Two of today's blocks overlap |
| `unscheduled_urgent` | Marked urgent, no time set aside |
| `integration_disconnected` | The account needs reauthorizing |
| `sync_failing` | A resource's last sync failed. Carries the provider's own message |

`not_enough_time` is the one the engine exists for. A deadline is not a problem because it is close; it is a problem when there is less free working time before it than the work needs. That is arithmetic, and it is computed with the same availability code the [planner](../../architecture/planner.md) uses, respecting working hours, working days and existing bookings. Work that already has a scheduled block is not flagged: the time has been set aside.

Every `detail` carries the numbers behind the claim. "This needs your attention" without a reason is just anxiety.

## Not here yet

Tasks repeatedly rolled over from one day to the next, which needs history Scalar does not record yet. Dismissing an item and having it stay dismissed.
