# Roadmap

Dates are not promised. Order is.

## Stage 1 (done)

- Repositories: `.github`, `docs`, `ui`, `sdk`, `api`, `web`, `website`.
- Magic link auth with real sessions (email delivery not implemented).
- Personal workspace, spaces, tasks, events, Today.
- Design tokens and first components in `ui`.
- Command palette (⌘K) with navigation and quick task capture.

## Stage 2 (current)

- Repositories added: `integrations`, `worker`.
- Google Calendar: OAuth with encrypted tokens, calendar discovery, incremental sync with `syncToken`, manual sync, disconnect with a keep or delete choice.
- `worker`: BullMQ consumer, retries with backoff, reconciliation scheduler.
- Integration settings in `web`; synced events visible in Today and Calendar.

Milestone 1 is met except for the AI answer: create an account, connect Google, sync the calendar, create a task, see both on Today.

## Next

- First AI command over read tools, through the provider abstraction ("What do I have tomorrow?").
- Gmail and Canvas integrations.

## Phase two

Milestones 2 and 3.

- Canvas: courses become Spaces, assignments become tasks, announcements enter Inbox.
- Gmail: messages enter Inbox; email to task.
- Inbox view and triage.
- Notifications.
- Search.
- Desktop shell (Tauri).

## Phase three

Milestone 4 and beyond.

- Scheduling proposals with approval before writing to the calendar.
- Calendar write, email send (approval always).
- Shared workspaces and roles.
- Mobile.
- More providers (Drive, Microsoft 365, others).
- Scalar Cloud, running the same code as self-hosted.

## Not planned

- A separate closed-source cloud edition.
- Autonomous actions on external systems without approval.
