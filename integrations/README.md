# Integrations

Two providers are implemented: Google Calendar (OAuth, events) and Canvas (personal access token, coursework). Both go through the same contract, which lives in `integrations/src/contract.ts`.

## Providers by phase

| Phase | Provider | Data | Direction |
| --- | --- | --- | --- |
| MVP | Google Calendar | events | read, later write with approval |
| Phase two | Canvas LMS | courses, assignments, announcements, modules, submissions | read |
| Phase two | Gmail | messages into Inbox | read, later send with approval |
| Phase three | Google Drive, Microsoft 365, Notion, others | files, calendar, mail | to be decided |

See [google.md](google.md) and [canvas.md](canvas.md).

## The ScalarIntegration contract

Every provider implements one interface, so the worker never names a provider to do its job:

```ts
interface ScalarIntegration {
  readonly id: ProviderId;
  readonly name: string;
  readonly auth: 'oauth2' | 'token';
  readonly capabilities: readonly IntegrationCapability[];
  readonly itemKinds: readonly ('event' | 'task')[];

  identify(input): Promise<{ externalAccountId: string; displayName: string | null }>;
  discover(input): Promise<IntegrationResource[]>;
  sync(context: SyncContext): Promise<SyncResult>;
}
```

The contract was generalized against two genuinely different providers rather than in the abstract. Generalising against Google alone would have produced something shaped exactly like Google: OAuth, calendars, sync tokens. Canvas has none of those, which is what makes it a useful second case.

What the contract deliberately does not assume: OAuth, calendars, that a provider can write, or that a provider has a cursor.

### Capabilities

`read_calendar`, `write_calendar`, `read_coursework`, `read_email`, `write_email`.

Providers declare what they can do and callers check. Neither implemented provider declares a write capability today: writing to someone's real calendar or sending mail on their behalf is a separate decision with its own approval story, and an absent capability is more honest than one declared and unimplemented.

### Normalized items

A sync returns `NormalizedEvent` (goes to `events`) or `NormalizedTask` (goes to `tasks`, with `status = inbox`). Both carry mandatory provenance so a synced row is traceable and a repeated sync updates rather than duplicates.

A `NormalizedTask` may carry `suggestedMinutes`, `suggestedPriority` and `suggestionReason`. Those are what the source *believes*, and they travel to the [inbox](../api/v1/inbox.md) as a suggestion a person accepts or turns down. They are never written straight onto a task.

### Adding a provider

A file in `integrations/src/<provider>/`, a line in `integrations/src/registry.ts`, and a value in the `integration_provider` enum. The worker and the API need no changes.

## Rules

- Minimum scopes. Request only what the current feature uses. Calendar read does not request calendar write. Adding a scope is a product decision and a re-consent for the user.
- Tokens are never logged and never returned to a client. See [../security/oauth-and-tokens.md](../security/oauth-and-tokens.md).
- External content is data, not instructions. See [../security/prompt-injection.md](../security/prompt-injection.md).
- Sync is cursor based, idempotent and retried with backoff. See [../architecture/sync.md](../architecture/sync.md).
- Every imported row carries provenance columns and a `source_url` back to the provider.
- Disconnecting an account revokes tokens and removes them; imported data is kept but marked as detached unless the user asks to delete it.
