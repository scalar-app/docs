# AI safety

Status: implemented. The rules on this page are enforced by [`@scalar/ai`](https://github.com/scalar-app/ai) and the `command` module in [`api`](https://github.com/scalar-app/api), and are covered by tests in both.

The one rule everything else follows from: **Scalar reads on its own, and never changes anything on its own.**

## Classification of actions

Every tool is classified when it is defined, and the classification decides what happens. Not the model's intent, not the wording of the request.

| Class | Meaning | Tools today | Behaviour |
| --- | --- | --- | --- |
| read | Reads workspace data the person can already see | `search_tasks`, `list_events`, `get_today`, `list_spaces`, `find_free_time` | Runs during the turn |
| suggest | Produces a proposal and changes nothing | none yet | Becomes a pending action |
| write | Changes Scalar or an external system | `create_task`, `update_task`, `schedule_task` | Never runs in the turn. Becomes a pending action |

There is no auto-approve, for any tool, in any workspace. External writes (calendar, email) will follow the same path when they land.

`defineTool` throws at construction if a `suggest` or `write` tool has no `describeAction`. An approval prompt can therefore never be blank: a tool that cannot explain itself cannot be registered.

## What happens in a turn

```text
Question -> model -> tool call -> classification
                                     |
              read ------------------+------------------ write or suggest
               |                                              |
        validate, execute,                            validate, record a
        return result to model                        pending action, tell
                                                      the model it has not
                                                      happened yet
```

The model is told, in the tool result itself, that a proposed change is *"awaiting their approval... It has not happened yet."* This is why the answer says "I can add that" rather than "I added that": the model is not guessing at the outcome, it has been told.

## Approval

A proposal is stored in `ai_actions` with status `pending`, the validated input, and the plain language summary the person will read. Approving is a separate HTTP request made by a person:

```text
POST /api/v1/command/actions/:id/approve
```

That handler re-checks authorization rather than trusting the turn that produced the proposal. A single conditional update matches on workspace, user and `pending` status at once, so:

- an action cannot be approved by anybody other than the person it was shown to,
- it cannot execute twice, because the second update matches no row,
- it cannot execute after being rejected.

The stored input is validated against the tool schema again before it reaches a service. If the tool no longer exists or the input no longer parses, the action is marked `failed` and nothing runs.

In the web app, Approve is never the default: it is not autofocused, Enter in the composer sends the next question rather than approving a pending card, and a decided card loses its buttons.

## The model never touches the database

Tools are executed by `api/src/modules/command/executors.ts`, which calls the same services the HTTP routes use. Every executor is scoped to the workspace on the tool context, which comes from the session and never from anything the model said. The model can name a task id; it cannot reach a task in another workspace, because the id is only ever used inside a workspace scoped query.

Tool definitions sent to the model contain name, description and JSON schema. Nothing else crosses that boundary.

## Refusals

When the provider declines, `stop_reason` is `refusal` and the turn ends there. No content blocks are read, no answer is invented, and the category is recorded on the message. The UI says it cannot help with that one and that nothing was changed.

## Determinism where determinism is possible

Free time is computed by `findFreeSlots` in `@scalar/ai`: overlapping events merged, working hours applied in the person's zone, past instants skipped, day boundaries walked in local time so daylight saving stays correct. The model asks for free time; it does not work it out. Asking a language model to do arithmetic that ordinary software does reliably produces a system that is wrong sometimes for no reason.

## External content is data

Text from email, calendar entries, course announcements and files is wrapped by `wrapExternalContent` and labelled as data that must never be treated as instructions. Delimiters are not a security control on their own, which is exactly why authorization lives in the tool layer and not in the prompt. A prompt injection that convinces the model to call `create_task` still produces nothing more than a card a person has to approve.

See [security/prompt-injection.md](../security/prompt-injection.md).

## What is recorded

`ai_messages` stores the text of each turn, the model, stop reason, token counts, and which tools ran with their classification and success. **Tool output is not stored.** Storing results would quietly turn the AI log into a second copy of a person's tasks and calendar, under a different retention story than the tables it was copied from.

`ai_actions` stores every proposal and its outcome, including ones that were rejected, so the record shows what was suggested and not only what happened.

## Provider abstraction

Model choice is configuration. `ScalarModelProvider` is the whole surface:

```ts
interface ScalarModelProvider {
  readonly name: string;
  generate(input: GenerateInput): Promise<GenerateResult>;
  stream?(input: GenerateInput): AsyncIterable<string>;
  structuredOutput<T>(input: StructuredOutputInput<T>): Promise<StructuredOutputResult<T>>;
  embed?(input: EmbedInput): Promise<EmbedResult>;
}
```

`AnthropicProvider` is the default. `ScriptedProvider` replays a fixed script and is what the test suites use, so no test needs an API key or makes a network call.

## Running without a model

`ANTHROPIC_API_KEY` is optional. Without it the API starts normally, Command returns 503 `AI_UNAVAILABLE`, and every other part of Scalar works. A half working AI feature is worse than a visibly absent one.

## Evaluation datasets

Not built yet. Prompt and tool changes will be checked against versioned datasets in `ai/evals/`: sample commands, expected tool calls, and expected refusals, including external content that instructs the model to delete tasks. Until then the behavioural guarantees are held by unit tests in `ai` and integration tests in `api`.

## Related

- [security/prompt-injection.md](../security/prompt-injection.md)
- [authorization.md](authorization.md)
- [deployment/what-it-costs.md](../deployment/what-it-costs.md)
