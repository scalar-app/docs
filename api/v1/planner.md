# Planner

Turns work that has no time into work that has one. Two endpoints: one that proposes and writes nothing, one that writes what a person approved.

The scheduling itself is a pure function in [scalar-app/ai](https://github.com/scalar-app/ai); see [architecture/planner.md](../../architecture/planner.md) for how it decides. This page is the HTTP contract. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/planner`).

## POST /api/v1/planner/preview

Body, all optional:

| Field | Meaning |
| --- | --- |
| `tz` | IANA zone. Defaults to the person's saved time zone; the SDK sends the reader's own |
| `rangeStart` | Defaults to now |
| `rangeEnd` | Defaults to seven days after `rangeStart`. At most 30 days |
| `taskIds` | Plan only these. Without it, every open task with no time is considered, up to 100 |

Response:

```json
{
  "rangeStart": "2026-08-19T08:00:00.000Z",
  "rangeEnd": "2026-08-26T08:00:00.000Z",
  "timeZone": "America/Los_Angeles",
  "blocks": [
    {
      "taskId": "9f…",
      "title": "Finish problem set",
      "startAt": "2026-08-19T16:00:00.000Z",
      "endAt": "2026-08-19T17:00:00.000Z",
      "minutes": 60,
      "reasons": ["due_within_24_hours", "fits_available_window"]
    }
  ],
  "unscheduled": [
    {
      "taskId": "2a…",
      "title": "Rewrite everything",
      "kind": "task_too_large_for_window",
      "detail": "This needs 600 minutes and the largest free block is 480."
    }
  ],
  "conflicts": [],
  "warnings": [{ "kind": "no_estimate_used_default", "taskId": "9f…", "detail": "…" }]
}
```

Preview writes nothing. A preview that changed the day would not be a preview.

What the planner treats as immovable: calendar events in the range, and tasks that already have a time. Re-planning work someone has already placed is a different feature and a louder one.

Errors: `422 INVALID_TIME_ZONE`, `422 INVALID_RANGE` (backwards), `422 RANGE_TOO_LARGE`, `422 TASK_NOT_IN_WORKSPACE`.

## POST /api/v1/planner/apply

```json
{
  "blocks": [
    { "taskId": "9f…", "startAt": "2026-08-19T16:00:00.000Z", "endAt": "2026-08-19T17:00:00.000Z" }
  ]
}
```

Response: `{ "applied": 1, "taskIds": ["9f…"] }`.

Apply takes the blocks back rather than re-running the planner. What a person approved is a specific set of times, not "whatever the planner says now", and sending them back is also what lets someone drop or adjust a block before accepting it.

Each block sets `scheduledStart` and `scheduledEnd` on its task. A task that was in the inbox becomes `todo`, because it has now been decided on; anything already in progress or blocked keeps its status.

**All or nothing.** The whole thing runs in one transaction. A half applied plan would leave a day in a state neither the person nor Scalar intended.

Errors:

| Code | Meaning |
| --- | --- |
| `409 PLAN_STALE` | The day changed since the preview: a task was completed, or an event now overlaps a proposed block. Nothing was written; preview again |
| `422 TASK_NOT_IN_WORKSPACE` | A task id does not belong to the caller's workspace |
| `422 VALIDATION_ERROR` | A block ends before it starts, or a task appears twice |

## What this is not

The planner does not write to external calendars. Scheduled work lives on Scalar's timeline; whether any of it should reach Google Calendar is a separate, opt-in question.

Nothing here is automatic. `autoSchedule` in [preferences](preferences.md) has an `apply` value reserved for a later phase, but today every plan is shown before it is written.
