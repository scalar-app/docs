# Timeline

One day as one ordered sequence. Calendar events and tasks that have been given a time are merged into blocks, so a person can read their day without holding two lists in their head.

Timeline is a read model. It has no table of its own and it schedules nothing: it composes what already exists. Deciding where a task should go is the planner's job, and the planner will read this to find out what is already there. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/timeline`), with the composition itself in `compute.ts` as a pure function.

## GET /api/v1/timeline

Query: `date` (YYYY-MM-DD, defaults to today in `tz`), `tz` (IANA zone, defaults to `UTC`). The SDK fills `tz` with the reader's own zone, because a timeline in UTC is the wrong day for almost everybody.

```json
{
  "date": "2026-08-19",
  "timeZone": "America/Los_Angeles",
  "busyMinutes": 195,
  "blocks": [
    {
      "id": "event:9f…",
      "itemId": "9f…",
      "blockType": "event",
      "title": "Calculus",
      "startAt": "2026-08-19T16:00:00.000Z",
      "endAt": "2026-08-19T17:15:00.000Z",
      "allDay": false,
      "locked": true,
      "source": "integration",
      "status": null,
      "priority": null,
      "spaceId": null,
      "projectId": null,
      "location": "Baskin 152"
    }
  ],
  "conflicts": [{ "blockIds": ["event:9f…", "task:2a…"] }]
}
```

- `id` is `event:<id>` or `task:<id>`. Stable within a day and derived from the item.
- `blockType` is `event` or `task`. `focus`, `personal` and `buffer` arrive with later phases.
- `locked` says whether Scalar may move the block. Calendar events are locked: Scalar does not get to decide someone is not in their lecture. Task blocks are movable.
- `source` is `manual`, `integration` or `planner`.
- `busyMinutes` counts minutes of the day covered by at least one timed block, counting overlaps once.
- `conflicts` reports pairs of timed blocks that overlap. It reports; it never resolves. Two lectures at the same hour is a fact about the day, not an error.

What is included: events overlapping the local day, and open tasks whose `scheduledStart` and `scheduledEnd` overlap it. A block that starts before midnight and runs into the day is included, and only the part inside the day counts toward `busyMinutes`.

What is excluded: tasks with no schedule (they are on Today by deadline instead), and tasks that are `done` or `cancelled`.

Ordering is all-day blocks first, then by start time, then by id, so two people reading the same day see the same order.

Errors: `422 INVALID_TIME_ZONE` for a zone this server does not know, `422 INVALID_DATE` for a date that is not a real calendar date.

## Planned

Focus sessions as their own block type, personal and buffer blocks, and multi-day ranges for a week view.
