# Spaces

A Space is a container inside a workspace: a course, a project, an area of life. Tasks may belong to one space or none. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/spaces`).

## Space shape

```json
{
  "id": "…",
  "workspaceId": "…",
  "name": "MATH 201",
  "description": null,
  "color": null,
  "archivedAt": null,
  "createdAt": "…",
  "updatedAt": "…"
}
```

## GET /api/v1/spaces

Lists spaces in the user's workspace, newest first. Query: `limit` (1 to 200, default 50), `cursor`. Response `{ "data": Space[], "nextCursor": string | null }`.

## POST /api/v1/spaces

Body: `name` (1 to 200 characters, required), `description` (up to 4000, nullable), `color` (up to 32, nullable). Response `201` with the Space.

## GET /api/v1/spaces/:id

Response: Space, or `404 NOT_FOUND` when the id does not exist in the caller's workspace.

## PATCH /api/v1/spaces/:id

Partial update of `name`, `description`, `color`, `archivedAt` (ISO datetime or `null`). At least one field is required. Response: updated Space.

Archiving is a soft state: set `archivedAt` to hide a space from active views without losing its tasks. Clear it with `null`.

## DELETE /api/v1/spaces/:id

Deletes the space. Tasks that referenced it get `spaceId = null`; they are not deleted. Response `200 { "ok": true }`.

## Planned

Spaces created automatically from Canvas courses and GitHub repositories, with provenance columns. Space detail pages in the web app.
