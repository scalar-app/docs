# Roadmap

Dates are not promised. Order is.

## Stage 1 (current)

- Repositories: `.github`, `docs`, `ui`, `sdk`, `api`, `web`, `website`.
- Magic link auth with real sessions (email delivery not implemented).
- Personal workspace, spaces, tasks, read-only events, Today.
- Design tokens and first components in `ui`.

## MVP

Milestone 1: create an account, connect Google, sync the calendar, create a task, see it on Today, ask "What do I have tomorrow?".

- Google Calendar integration (read).
- `worker` with cursor-based sync, retries, dead-letter.
- Command palette (⌘K).
- First AI command over read tools, through the provider abstraction.
- Email delivery for magic links.

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
