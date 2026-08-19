# Focus

One stretch of work on one task. Focus reduces the system to a single thing, and the record it leaves exists to make future estimates better and to give someone their own history.

It is not a productivity score, it is not visible to anyone but the person who did the work, and the learning half of it can be turned off without losing the record. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/focus`).

## Session shape

```json
{
  "id": "…",
  "taskId": "…",
  "taskTitle": "Finish problem set",
  "status": "active",
  "plannedMinutes": 45,
  "startedAt": "2026-08-19T16:00:00.000Z",
  "endedAt": null,
  "actualMinutes": null,
  "notes": null
}
```

`status` is `active`, `completed` or `cancelled`.

Sessions belong to the person, not the workspace. Another member of a shared workspace cannot see, end, or list them.

## GET /api/v1/focus/current

The running session, or `{ "session": null }`. No session is a normal state rather than an error, so this never 404s.

## POST /api/v1/focus/start

Body: `taskId` (required), `plannedMinutes` (1 to 480, optional).

`plannedMinutes` falls back to the task's estimate, then to `defaultFocusMinutes` from [preferences](preferences.md).

Starting a session sets the task to `in_progress`, because that is what starting work means.

Errors: `409 FOCUS_IN_PROGRESS` (focus is one thing at a time, and a partial unique index enforces that in the database rather than trusting callers), `404 NOT_FOUND`, `422 TASK_CLOSED` for work that is already done or cancelled.

## POST /api/v1/focus/:id/complete

Body: `notes` (optional), `completeTask` (optional).

```json
{
  "session": { "…": "…", "status": "completed", "actualMinutes": 47 },
  "taskCompleted": false,
  "estimateUpdated": true,
  "typicalMinutes": null
}
```

- `actualMinutes` is wall clock time, minimum one.
- `completeTask` finishes the work itself, not just the session. Ending a session without it leaves the task in progress.
- `estimateUpdated` says Scalar filled in an estimate that was empty. **It never overwrites an estimate a person set**: someone who said an hour meant an hour, and replacing that with what actually happened would make their own numbers untrustworthy. The flag exists so a changed estimate is something they are told about rather than something they discover.
- `typicalMinutes` is the mean of finished sessions on that task, and is null until there are at least three. Two sessions is not a pattern.

With `durationLearningEnabled` off in preferences, no estimate is filled and `typicalMinutes` is always null. The session itself is still recorded, because it is the person's own history either way.

Errors: `409 FOCUS_ALREADY_ENDED`, `404 NOT_FOUND`.

## POST /api/v1/focus/:id/cancel

Ends a session without recording it as work done: no `actualMinutes`, nothing learned. For a run that was abandoned rather than finished.

## GET /api/v1/focus/sessions

The person's own history, newest first. Query: `limit`, `cursor`, `taskId`.

## Planned

Focus sessions as their own block type on the [timeline](timeline.md), and estimates informed by sessions on similar work rather than only the same task.
