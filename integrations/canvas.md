# Canvas

Coursework from Canvas LMS: courses become resources, assignments arrive in your inbox with what Canvas believes about them attached as a suggestion.

Read only. Scalar does not submit work.

## Why a token rather than OAuth

Canvas OAuth requires a developer key, and only an institution's Canvas administrator can issue one. A student cannot get one, and a self hoster should not have to email their university to use their own assignments.

So Canvas connects with a **personal access token**, which anyone can generate for themselves:

1. In Canvas: **Account**, then **Settings**.
2. Under Approved Integrations, **+ New access token**. Give it a purpose and leave the expiry blank, or set one and expect to reconnect.
3. Copy the token: Canvas shows it once.
4. In Scalar: **Settings**, then **Integrations**, then Canvas. Enter your institution's Canvas address and the token.

The token is verified against Canvas before anything is stored, so a typo fails immediately with a message about the token rather than as a mysterious sync error later. It is then encrypted at rest with your server's `TOKEN_ENCRYPTION_KEY`, and no endpoint ever returns it.

## What syncs

| Canvas | Scalar |
| --- | --- |
| Active course | A **Project**, and a sync resource |
| Published assignment | A task in your **inbox**, with `dueAt`, a link back, and a plain-text description |
| Points possible | A *suggested* duration and priority, with the reasoning |

A course becomes a project keyed by Canvas's own course id, so it is the same project on every sync and a course that gets renamed in Canvas is renamed in Scalar rather than duplicated. Assignments are filed into it when they first arrive.

What the provider owns is the name. **Everything else about the project is yours**: which Space it sits in, its status, its dates. A sync never touches those, and it never moves an assignment you have re-filed somewhere else. Once you put a course into a Space, work that arrives later is filed into that Space too.

Concluded courses and unpublished assignments are skipped. On a first sync, coursework due more than 30 days ago is skipped as history; after that an assignment Scalar already knows about keeps updating even once its deadline passes.

## What Scalar decides and what you decide

The provider says what exists and when it is due. **It does not get to say what matters.**

An imported assignment lands with `status = inbox`, no priority and no estimate. What Canvas believes about size travels separately as an inbox [suggestion](../api/v1/inbox.md):

> Homework 4 — Worth 60 points in Canvas.
> Suggested: 2 hr, high priority. **Accept** / **Not this**

Points are a weak signal: a 100 point final and a 100 point participation grade are not the same afternoon. So it is advice with its reasoning attached, and you can change it before accepting.

## Repeated syncs

Every assignment carries a stable `sourceObjectId` (`course:<id>:assignment:<id>`), and tasks are unique on `(integration_account_id, source_object_id)`. Syncing twice updates one task rather than creating two.

A sync refreshes what the provider owns — title, description, due date, link — and **never touches what you own**: status, priority, estimate, space, project and schedule. Someone who decided an assignment is urgent and blocked out Thursday for it does not lose that because Canvas edited a description. A suggestion is attached once, when the task first appears, so advice you have turned down does not come back.

Canvas does not report deletions, so an assignment that disappears from Canvas stays in Scalar. Delete it yourself if you want it gone.

## Limits

- Canvas has no sync cursor, so each run fetches the course's assignments and compares. That is affordable for tens of assignments per course and honest: pretending to be incremental against an API that is not would silently miss edits.
- Announcements, submissions, grades and calendar events are not imported.
- One token, one Canvas. Two institutions means two accounts.
