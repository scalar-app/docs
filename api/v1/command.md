# Command

Scalar Command: ask a question about your own tasks and calendar, and approve anything that would change them.

Every endpoint here requires a session. On a server with no model key configured they all return `503` with code `AI_UNAVAILABLE`; treat that as "hide the feature", not as an outage. See [../../architecture/ai-safety.md](../../architecture/ai-safety.md) for the rules these endpoints enforce.

## POST /api/v1/command

Runs one turn.

Request:

```json
{
  "message": "block two hours for the problem set tomorrow",
  "threadId": "0b2f...",
  "timeZone": "America/New_York"
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `message` | yes | 1 to 4000 characters after trimming |
| `threadId` | no | Continues a thread. Omit to start a new one |
| `timeZone` | no | IANA zone, defaults to `UTC`. Answers use it |

Response:

```json
{
  "threadId": "0b2f...",
  "messageId": "77a1...",
  "answer": "Thursday morning is clear. I can block 09:00 to 11:00.",
  "actions": [
    {
      "id": "9c31...",
      "tool": "schedule_task",
      "classification": "write",
      "summary": "Schedule work from 2026-03-12 09:00 to 2026-03-12 11:00",
      "status": "pending",
      "createdAt": "2026-03-11T18:04:02.113Z"
    }
  ],
  "stopReason": "needs_approval",
  "refusalCategory": null,
  "usage": { "inputTokens": 2104, "outputTokens": 188 }
}
```

`stopReason` is one of:

| Value | Meaning |
| --- | --- |
| `answered` | The question was answered. Nothing is pending |
| `needs_approval` | One or more actions are waiting on a decision |
| `refused` | The provider declined. `answer` is empty and `refusalCategory` is set |
| `max_steps` | The loop hit its step limit without settling |
| `max_tokens` | The answer was cut short |

**Actions in the response have not happened.** Read tools run during the turn; anything that would write is recorded as `pending` and executes only through the approve endpoint.

Errors: `422` for an invalid body, `404` for an unknown `threadId`, `429` when the caller passes `AI_DAILY_MESSAGE_LIMIT`, `503` when no model is configured or the provider is unreachable.

## POST /api/v1/command/actions/:id/approve

Executes one pending action. This is the only endpoint in Scalar that turns a proposal into a change.

Response:

```json
{
  "action": { "id": "9c31...", "tool": "schedule_task", "status": "executed", "...": "..." },
  "resultId": "4f0a...",
  "error": null
}
```

`resultId` is the id of the task that was created or changed. A failed execution is a `200` with `status: "failed"` and an `error` message, because the request succeeded and the outcome is something to show. Reserve error statuses for actual request failures.

| Status | When |
| --- | --- |
| `404` | No such action, or it belongs to somebody else |
| `403` | Already approved, rejected or executed |

Authorization is re-checked here rather than trusted from the turn that produced the proposal. Approving is idempotent in the safe direction: the second attempt is refused, not repeated.

## POST /api/v1/command/actions/:id/reject

Declines a pending action. Returns the action with status `rejected`. Same `404` and `403` behaviour as approve.

## GET /api/v1/command/threads

Cursor paginated, newest first. Scoped to the workspace and to the person who started the threads.

```json
{
  "data": [
    {
      "id": "0b2f...",
      "title": "block two hours for the problem set tomorrow",
      "createdAt": "2026-03-11T18:03:58.201Z",
      "lastMessageAt": "2026-03-11T18:04:02.980Z"
    }
  ],
  "nextCursor": null
}
```

The title is taken from the opening message, so a thread list reads in the person's own words.

## GET /api/v1/command/threads/:id

One thread with its messages, each carrying the actions it proposed.

```json
{
  "id": "0b2f...",
  "title": "add a task to email the TA",
  "createdAt": "2026-03-11T18:03:58.201Z",
  "lastMessageAt": "2026-03-11T18:04:02.980Z",
  "messages": [
    {
      "id": "77a0...",
      "role": "user",
      "content": "add a task to email the TA",
      "stopReason": null,
      "refusalCategory": null,
      "createdAt": "2026-03-11T18:03:58.201Z",
      "actions": []
    },
    {
      "id": "77a1...",
      "role": "assistant",
      "content": "I can add that for you.",
      "stopReason": "needs_approval",
      "refusalCategory": null,
      "createdAt": "2026-03-11T18:04:02.113Z",
      "actions": [{ "id": "9c31...", "status": "pending", "...": "..." }]
    }
  ]
}
```

`404` if the thread does not exist or belongs to another workspace or person.

## What is stored

Message rows record the text, model, stop reason, token counts, and which tools ran with their classification and whether they succeeded. Tool output is not stored, so an AI message never becomes a second copy of your tasks and calendar.
