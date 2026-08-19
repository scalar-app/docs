# Projects

A Project is a body of work inside a Space: a course assignment series, a release, a piece of writing. Tasks may belong to one project or none. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/projects`).

The hierarchy is Workspace, then Space, then Project, then Task. Workspace is the tenancy boundary and does not change; Space and Project are how a person organises their own work.

## Project shape

```json
{
  "id": "…",
  "workspaceId": "…",
  "spaceId": "…",
  "name": "CSE 13S",
  "description": null,
  "status": "active",
  "startAt": null,
  "dueAt": "2026-12-10T00:00:00.000Z",
  "createdBy": "…",
  "createdAt": "…",
  "updatedAt": "…"
}
```

`status` is one of `active`, `paused`, `completed`, `archived`.

`source` is `scalar` for a project you made, or the provider it mirrors. A Canvas course becomes a project with `source: "canvas"`; sync keeps its name in step and leaves everything else to you. See [integrations/canvas.md](../../integrations/canvas.md).

## GET /api/v1/projects

Lists projects in the caller's workspace, newest first. Query: `limit` (1 to 200, default 50), `cursor`, `status` (comma separated), `spaceId`. Response `{ "data": Project[], "nextCursor": string | null }`.

## POST /api/v1/projects

Body: `name` (1 to 200 characters, required), `description` (up to 20000, nullable), `spaceId` (nullable), `status`, `startAt`, `dueAt` (ISO datetimes, nullable). Response `201` with the Project.

`dueAt` before `startAt` is rejected with `422 VALIDATION_ERROR`. A `spaceId` from another workspace is rejected with `422 SPACE_NOT_IN_WORKSPACE`.

## GET /api/v1/projects/:id

Response: Project, or `404 NOT_FOUND` when the id does not exist in the caller's workspace.

## PATCH /api/v1/projects/:id

Partial update of the same fields. At least one is required. Response: updated Project.

## DELETE /api/v1/projects/:id

Deletes the project. Tasks that referenced it get `projectId = null`; they are not deleted, on the principle that the work outlives the container it was filed in. Response `200 { "ok": true }`.

## Tasks

Tasks carry `projectId` (nullable) and `GET /api/v1/tasks` accepts `projectId` as a filter. A project id from another workspace is rejected with `422 PROJECT_NOT_IN_WORKSPACE`.

## Planned

Project detail views in the web app, and projects created automatically from Canvas courses.
