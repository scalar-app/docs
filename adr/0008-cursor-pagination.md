# 0008. Cursor based pagination

Date: 2026-08-18
Status: Accepted

## Context

Lists in Scalar (tasks, spaces, events, later inbox items) change constantly while a user pages through them, and sync workers insert rows behind the user's back. Offset pagination skips or repeats rows under those conditions and gets slower as offsets grow.

## Decision

Every list endpoint accepts `?limit=&cursor=` and returns `{ data, nextCursor }`. The cursor is an opaque base64url encoding of the sort timestamp of the last row plus its id as a tiebreaker (keyset pagination). Each endpoint documents its sort key: `createdAt` descending for tasks and spaces, `startsAt` ascending for events. `limit` is 1 to 200, default 50.

## Consequences

- Stable paging under concurrent writes and constant cost per page, using the composite indexes that already exist for the sort keys.
- Clients cannot jump to page N. Scalar's interfaces do not need that.
- Cursors are opaque; clients must not parse them. `INVALID_CURSOR` is returned when one is malformed.
- The SDK provides `paginate()` and `collectAll()` helpers so callers rarely handle cursors by hand.

## Alternatives considered

- Offset and limit: simple, but incorrect under writes and slow at depth.
- Page numbers with total counts: requires expensive counts and gives the same skipping problem.
