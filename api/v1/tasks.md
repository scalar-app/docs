# Tasks

All routes require a session and operate on the user's workspace.

## Task shape

```json
{
  "id": "…",
  "workspaceId": "…",
  "spaceId": null,
  "title": "Study Calculus",
  "description": null,
  "status": "todo",
  "priority": "medium",
  "dueAt": "2026-08-20T16:00:00.000Z",
  "scheduledStart": null,
  "scheduledEnd": null,
  "estimatedMinutes": 90,
  "sourceId": null,
  "parentTaskId": null,
  "createdBy": "…",
  "createdAt": "…",
  "updatedAt": "…",
  "completedAt": null
}
```

Enums:

- `status`: `inbox | todo | in_progress | blocked | done | cancelled`
- `priority`: `none | low | medium | high | urgent`

`completedAt` is set by the server when `status` becomes `done`.

## GET /api/v1/tasks

Query parameters:

| Param | Type | Meaning |
| --- | --- | --- |
| `status` | comma list of status | Filter by one or more statuses |
| `spaceId` | uuid | Tasks in one space |
| `dueBefore` | ISO timestamp | `dueAt <= dueBefore` |
| `dueAfter` | ISO timestamp | `dueAt >= dueAfter` |
| `q` | string | Case-insensitive substring match on `title` (`ILIKE`) |
| `limit`, `cursor` | | See [conventions](../conventions.md) |

Response: `{ "data": Task[], "nextCursor": string | null }`, newest first (`createdAt` desc).

## POST /api/v1/tasks

Body: `title` required; every other writable field optional (`spaceId`, `description`, `status`, `priority`, `dueAt`, `scheduledStart`, `scheduledEnd`, `estimatedMinutes`, `parentTaskId`). Defaults: `status = inbox`, `priority = none`. `title` is 1 to 500 characters, `description` up to 20000, `estimatedMinutes` 0 to 100000. When both `scheduledStart` and `scheduledEnd` are given, end must be after start.

`spaceId` and `parentTaskId` must belong to the same workspace, otherwise `422 SPACE_NOT_IN_WORKSPACE` or `422 PARENT_TASK_NOT_IN_WORKSPACE`.

Response: `201` with the Task.

## GET /api/v1/tasks/:id

Response: Task, or `404 NOT_FOUND`.

## PATCH /api/v1/tasks/:id

Partial update of writable fields; at least one field is required. Send `null` to clear a nullable field. Setting `status` to `done` sets `completedAt`; moving away from `done` clears it. A task cannot be its own parent (`422 INVALID_PARENT_TASK`). Response: updated Task.

## DELETE /api/v1/tasks/:id

Deletes the task. Response `200 { "ok": true }`. Stage 1 hard deletes; soft delete is planned alongside audit logs.

## Not in Stage 1

- Task dependencies (`task_dependencies` table is planned).
- Bulk operations.
- Full text search beyond `q`.
