# Canvas LMS

Status: planned (Phase two). Nothing is implemented.

## What the Canvas REST API provides

Canvas exposes a REST API that, for a student token, realistically gives Scalar:

| Resource | Endpoint family | What Scalar does with it |
| --- | --- | --- |
| Courses | `/api/v1/courses` (enrolled, active) | One Space per course |
| Assignments | `/api/v1/courses/:id/assignments` | One task per assignment, `dueAt` from `due_at`, `sourceUrl` from `html_url` |
| Submissions | `/api/v1/courses/:id/students/submissions` | Mark task done when submitted, show grade if present |
| Announcements | `/api/v1/announcements?context_codes[]=course_:id` | Inbox items |
| Modules and items | `/api/v1/courses/:id/modules` | Optional: structure inside a Space |
| Calendar events | `/api/v1/calendar_events` | Course events with dates |
| Planner | `/api/v1/planner/items` | Cross-course "what is due" list, useful as a cheap incremental feed |

Things Canvas does not give reliably, and Scalar will not promise:

- Push notifications. There are no webhooks for student tokens; Scalar polls.
- A single change cursor. Canvas has no global sync token. Incremental sync uses per-course listing with `updated_since` style filters where they exist, plus periodic full listing for reconciliation. See [../architecture/sync.md](../architecture/sync.md).
- Consistent due dates. Assignments can have overrides per section; `due_at` may be null. Scalar takes the effective due date when the API returns it and otherwise leaves `dueAt` null.
- Grades for everything. Some courses hide grades; Scalar shows what the API returns and nothing else.
- Files beyond metadata. Downloading course files is out of scope initially.

Institutions can restrict or disable API access. When that happens Scalar shows the provider error and does not guess.

## Authentication

Two token types, chosen by deployment:

- Canvas developer key (OAuth2). Requires an admin at the institution to create a developer key for Scalar. Suitable for a hosted deployment serving many users at one school. Tokens are refreshable.
- Personal access token. Any Canvas user can generate one under Account, Settings, "New Access Token". Suitable for self-hosters and for development. Not refreshable; the user pastes it and can revoke it in Canvas.

Both are stored encrypted in `integration_tokens` and never shown again after entry. The user also supplies the Canvas base URL (`https://school.instructure.com`), since every institution has its own.

## Rate limits

Canvas applies per-token throttling with `X-Rate-Limit-Remaining` and a cost model. The adapter reads these headers and sets `next_sync_at` accordingly. Pagination is via `Link` headers.

## Mapping summary

| Canvas | Scalar |
| --- | --- |
| course | space (`source_provider = canvas`) |
| assignment | task, status `todo`, `dueAt = due_at` |
| submitted submission | task status `done`, `completedAt = submitted_at` |
| announcement | inbox item |
| calendar event | event (`source = canvas`) |
