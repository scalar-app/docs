# Preferences

Personal settings that describe someone's day. These are the planner's inputs: without them a scheduler has to invent working hours, and inventing them is how scheduling software ends up proposing three in the morning.

Preferences belong to the person, not the workspace. Switching context does not change when someone works. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/preferences`).

## Preferences shape

```json
{
  "timeZone": "America/Los_Angeles",
  "weekStartsOn": 1,
  "workdayStartMinute": 540,
  "workdayEndMinute": 1020,
  "workDays": [1, 2, 3, 4, 5],
  "defaultFocusMinutes": 50,
  "minimumBufferMinutes": 10,
  "autoSchedule": "suggest",
  "durationLearningEnabled": true,
  "updatedAt": null
}
```

- `timeZone`: IANA zone. Every planner decision is made in this zone, never the server's.
- `weekStartsOn` and `workDays`: ISO weekdays, 1 is Monday.
- `workdayStartMinute` and `workdayEndMinute`: minutes from local midnight, so 540 is 09:00.
- `autoSchedule`: `off`, `suggest` or `apply`. Only `suggest` is used today; a plan is always shown before anything is written.
- `durationLearningEnabled`: whether finished focus sessions may inform future estimates. Off keeps the sessions as personal history and draws nothing from them.
- `updatedAt`: null for someone who has never changed anything, which is how a caller tells the server's defaults apart from a deliberate choice that happens to match them.

## GET /api/v1/preferences

Returns the caller's preferences, or the defaults above when they have never saved any. Never 404s.

## PATCH /api/v1/preferences

Partial update. At least one field is required. The patch is merged onto what is stored and the result is validated, because a working day is only coherent as a pair: sending a start alone can still invert it. Response: the saved preferences.

Errors:

| Code | Meaning |
| --- | --- |
| `422 VALIDATION_ERROR` | A field is out of range, or `timeZone` is not a zone this server knows |
| `422 INVALID_WORKING_HOURS` | The merged working day would end at or before it starts |

`workDays` is deduplicated and sorted on the way in, so the stored value has one shape.
