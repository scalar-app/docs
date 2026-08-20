# Inbox

Where external chaos enters Scalar, and where it gets decided.

**There is no inbox table.** An unfiled task is a task with `status = 'inbox'`; giving unfiled work its own table would mean two places to look for the same thing. What this module adds is the envelope around it: something arriving from Canvas or Gmail can carry a proposal about where it belongs, and a person can accept, edit, or turn that proposal down. Implemented in [scalar-app/api](https://github.com/scalar-app/api) (`src/modules/inbox`).

**Nothing in a suggestion is applied on its own.**

## GET /api/v1/inbox

Unfiled work with whatever is proposed about it, in one call: triage is one screen.

```json
{
  "data": [
    {
      "task": { "id": "9f…", "title": "Homework 4", "status": "inbox", "…": "…" },
      "suggestion": {
        "id": null,
        "origin": "planner",
        "source": "scalar",
        "reason": "Fits before it is due, in 90 minutes of free working time.",
        "values": {
          "scheduledStart": "2026-08-20T16:00:00.000Z",
          "scheduledEnd": "2026-08-20T17:30:00.000Z",
          "estimatedMinutes": 90
        }
      }
    }
  ],
  "nextCursor": null
}
```

Suggestions come from two places:

- **Stored** (`id` is set, `origin: "integration"`). Written by an integration into `task_suggestions`. Canvas and Gmail arrive in a later phase; the table and its contract exist now so they have somewhere coherent to land.
- **Worked out on the spot** (`id` is null, `origin: "planner"`). For an item with a deadline and no time yet, Scalar asks the [planner](../../architecture/planner.md) where it would fit and offers that. It is computed per request and never written.

A stored suggestion wins over a computed one. An item with nothing useful to say about it has `suggestion: null`, which is the normal case for a quick capture with no deadline.

## POST /api/v1/inbox/:taskId/accept

```json
{ "values": { "spaceId": "…", "estimatedMinutes": 30 }, "suggestionId": "…" }
```

Applies the values and files the item: an item in the inbox becomes `todo`, because accepting is the decision that takes it out.

Accept takes the values *back*, the way [applying a plan](planner.md) does. What a person agreed to is what was on screen, which may not be what was suggested, because editing before accepting is the point.

`suggestionId` is optional and only meaningful for a stored suggestion. Given one, the row is marked `accepted` when the values match what was proposed and `edited` when they do not: how good a suggestion was is a different fact from whether it was used.

Errors: `404 NOT_FOUND`, `422 SPACE_NOT_IN_WORKSPACE`, `422 PROJECT_NOT_IN_WORKSPACE`, `422 INVALID_SCHEDULE`, `422 SUGGESTION_NOT_FOR_TASK`.

## POST /api/v1/inbox/:taskId/dismiss

Turns down the proposal. **The item stays in the inbox** and is unchanged: the decision was about the advice, not the work. A dismissed stored suggestion is not offered again.

To get rid of the work itself, cancel the task through `PATCH /api/v1/tasks/:id`, which marks it cancelled rather than deleting it, so nothing is lost.

## Writing suggestions

`InboxService.suggest()` is the contract integrations write through. There is deliberately no HTTP endpoint for it yet: nothing calls it until Canvas and Gmail land, and an endpoint with no caller is a guess about what they will need.

## Planned

Suggested space and project from the source (a Canvas course becomes a project), suggested durations from focus history on similar work, and converting an email into a task with the message linked.
