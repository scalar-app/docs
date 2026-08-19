# Search

One request across everything a person can see in a workspace.

Requires a session.

## GET /api/v1/search

| Parameter | Required | Notes |
| --- | --- | --- |
| `q` | yes | 2 to 200 characters after trimming. Shorter is rejected with `422` |
| `limit` | no | How many of each kind to return, 1 to 50, default 10. Not a total |

```json
{
  "query": "problem",
  "tasks": [{ "id": "...", "title": "Finish problem set 4", "...": "..." }],
  "events": [],
  "spaces": [{ "id": "...", "name": "Problem sets", "...": "..." }],
  "counts": { "tasks": 1, "events": 0, "spaces": 1, "total": 2 }
}
```

Tasks, events and spaces are full DTOs, the same shapes their own endpoints return, so a caller
can render a result without a second request.

`limit` is per kind rather than a total, so one noisy kind cannot crowd the others out of the
response.

## What it matches

Substring matching with `ilike`, case insensitive, on:

| Kind | Fields |
| --- | --- |
| Tasks | title, description |
| Events | title, description, location |
| Spaces | name, description |

This is not ranked full text search, and the ordering is recency rather than relevance: most
recently updated tasks, most recent events, most recently created spaces. Saying so is more
useful than implying a ranking that does not exist. It suits a personal workspace, where row
counts are in the thousands, and it needs no index to maintain and no extension to install.

LIKE wildcards are escaped, so `%` and `_` are searched for literally. Without that, a search for
`%` would match every row and quietly turn the search box into a data dump.

Archived spaces are excluded, since they are not somewhere you are still working.

## Inbox

There is no inbox endpoint. Anything captured without being filed carries the `inbox` task
status, so the inbox is:

```text
GET /api/v1/tasks?status=inbox
```

and triage is an ordinary task update: `PATCH /api/v1/tasks/:id` with a new status, and a
`spaceId` when filing it somewhere. A separate table would mean two places to look for the same
thing.
