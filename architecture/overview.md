# Architecture overview

## The concept

Everything that demands a person's attention enters Scalar: email, calendar events, Canvas assignments, tasks, files, notifications. Scalar does three things with that stream:

1. Understands it: what it is, where it came from, when it matters.
2. Organizes it: into Spaces, tasks, a calendar and a single Today view.
3. Helps act on it: turning an email into a task, proposing a schedule, answering "What do I have tomorrow?".

External systems remain the source of record where they already are one (Google Calendar, Canvas). Scalar stores provenance for every imported object so the user can always go back to the origin.

## Primary navigation

| Item | Purpose |
| --- | --- |
| Today | Deterministic daily view: urgent, overdue, due today, upcoming events |
| Inbox | Items that entered Scalar and have not been triaged yet |
| Tasks | All tasks, filterable by status, space and due date |
| Calendar | Events, native and synced |
| Spaces | Containers for a course, project or area of life |
| Search | Full text across tasks, events, inbox items |
| Command (⌘K) | Keyboard-first entry point for navigation, creation and AI commands |

Stage 1 implements Today, Tasks and Spaces on top of the API. Inbox, Calendar (write side), Search and AI commands are planned.

## Repository relationship

```text
                +----------+   +----------+   +----------+
                |   web    |   |  mobile  |   | desktop  |
                | Next.js  |   | (later)  |   |  Tauri   |
                +----+-----+   +----+-----+   +----+-----+
                     |              |              |
                     +--------------+--------------+
                                    |  HTTP /api/v1
                                    v
                              +-----------+
                              |    api    |
                              |  Fastify  |
                              +-----+-----+
                                    |
                                    v
                             +-------------+
                             | PostgreSQL  |
                             +-------------+
                                    ^
                     +--------------+--------------+
                     |                             |
               +-----+------+               +------+-----+
               |   worker   |               |     ai     |
               | jobs, sync |               | providers, |
               +-----+------+               | tools      |
                     |                      +------------+
                     v
             +----------------+
             |  integrations  |
             | Google, Canvas |
             +----------------+

   shared:  sdk (typed client)   ui (design system, tokens)
   support: infra (compose, deploy)   docs (this repo)   .github (org profile)
```

Clients talk only to `api`. `worker` and `ai` share the database with `api` and run out of process. `worker` is the only component that calls external providers.

## Milestones

1. Account, Google connection, calendar sync, create a task, see it on Today, ask "What do I have tomorrow?".
2. Connect Canvas, courses become Spaces, assignments become tasks with due dates.
3. Connect Gmail, messages land in Inbox, turn an email into a task.
4. Scheduling proposals: Scalar suggests when to do a task and asks for approval before writing to the calendar.

Stage 1 covers the account, task, space and Today parts of milestone 1. Google connection and calendar sync are not implemented yet.

## Related pages

- [repositories.md](repositories.md)
- [data-model.md](data-model.md)
- [sync.md](sync.md)
- [ai-safety.md](ai-safety.md)
- [authorization.md](authorization.md)
