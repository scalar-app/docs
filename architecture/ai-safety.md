# AI safety

Status: planned. Stage 1 has no AI code. This page fixes the rules the `ai` repository must follow when it exists.

## Classification of actions

Every tool the model can call is classified before it is registered:

| Class | Meaning | Examples | Approval |
| --- | --- | --- | --- |
| read | Reads Scalar data for the current workspace | list tasks, get events for a range, search | none |
| suggest | Produces a proposal that the user may accept | propose a schedule, draft a task from an email | none to produce, explicit accept to apply |
| write | Mutates Scalar data or an external system | create task, move task, update event, send email | explicit approval every time, unless the user has enabled auto-approve for that specific tool in that workspace |

External writes (calendar, email) always require approval. There is no auto-approve for them.

## Approval UI

Write actions are shown as a plain statement of intent before anything happens:

```text
Scalar wants to:
  Move "Study Calculus" from Tue 14:00 to Wed 09:00 (Google Calendar: Personal)

  [Approve]  [Edit]  [Deny]
```

Approving records an `ai_actions` row with the exact payload that was executed. Denying records the denial. Editing opens the payload for the user to change before approval.

## Tool architecture

```text
User -> Command -> Model -> Tool request -> Authorization -> Validation -> Execution or Approval -> Result
```

1. User issues a command (Command palette, chat, or a button).
2. Model receives the command plus a tool list and produces a tool request as structured output.
3. Authorization: the request is checked against the current user's workspace membership and permissions, exactly as an HTTP request would be. See [authorization.md](authorization.md).
4. Validation: the payload is parsed with a zod schema. Anything that does not parse is rejected and the model is told why.
5. Execution for read tools; approval flow for write tools. Both go through the same service layer as the API.
6. Result is returned to the model, and to the user.

The model never touches the database. It only ever sees tool results and only ever produces tool requests. Tool implementations live in `api` service code and are called by the `ai` runtime.

## Tools (planned initial set)

| Tool | Class |
| --- | --- |
| `tasks.list`, `tasks.get`, `tasks.search` | read |
| `events.list` | read |
| `spaces.list` | read |
| `today.get` | read |
| `inbox.list` | read |
| `schedule.propose` | suggest |
| `tasks.create`, `tasks.update`, `tasks.complete` | write |
| `events.create`, `events.update` (external) | write, approval always |
| `inbox.toTask` | write |

## Provider abstraction

The `ai` repository wraps model providers behind one interface so that provider choice is configuration:

```ts
interface AIProvider {
  generate(input: GenerateInput): Promise<GenerateOutput>;
  stream(input: GenerateInput): AsyncIterable<StreamChunk>;
  embed(input: EmbedInput): Promise<EmbedOutput>;
  structuredOutput<T>(input: GenerateInput, schema: ZodType<T>): Promise<T>;
}
```

`structuredOutput` is how tool requests are produced: the schema is the union of tool payload schemas.

## Evaluation datasets

Prompt and tool changes are checked against versioned datasets in `ai/evals/`: sample commands, expected tool requests, and expected refusals (for example, external content instructing the model to delete tasks must not produce a delete request). Evals run in CI for the `ai` repository.

## Related

- [security/prompt-injection.md](../security/prompt-injection.md)
- [authorization.md](authorization.md)
