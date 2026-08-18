# Integrations

Status: planned. No integration code exists in Stage 1. This section defines what will be built and the rules it must follow.

## Providers by phase

| Phase | Provider | Data | Direction |
| --- | --- | --- | --- |
| MVP | Google Calendar | events | read, later write with approval |
| Phase two | Canvas LMS | courses, assignments, announcements, modules, submissions | read |
| Phase two | Gmail | messages into Inbox | read, later send with approval |
| Phase three | Google Drive, Microsoft 365, Notion, others | files, calendar, mail | to be decided |

See [google.md](google.md) and [canvas.md](canvas.md).

## The ScalarIntegration interface

Every provider adapter implements one interface so the worker can drive them uniformly:

```ts
interface ScalarIntegration {
  readonly provider: string;                     // 'google', 'canvas'
  readonly scopes: readonly string[];            // minimum scopes needed
  authorize(input: AuthorizeInput): Promise<AuthorizeResult>;   // OAuth start / token exchange, or PAT validation
  refresh(tokens: StoredTokens): Promise<StoredTokens>;
  revoke(tokens: StoredTokens): Promise<void>;
  listSources(ctx: IntegrationContext): Promise<Source[]>;      // calendars, courses
  sync(ctx: IntegrationContext, source: Source, cursor: string | null): Promise<SyncResult>;
  // SyncResult: { objects: SourceObject[]; deletedIds: string[]; nextCursor: string | null; rateLimit?: RateLimitInfo }
}
```

Adapters return raw `SourceObject`s. Mapping into tasks, events and inbox items happens in shared code, so provenance and dedup rules are applied once.

## Rules

- Minimum scopes. Request only what the current feature uses. Calendar read does not request calendar write. Adding a scope is a product decision and a re-consent for the user.
- Tokens are never logged and never returned to a client. See [../security/oauth-and-tokens.md](../security/oauth-and-tokens.md).
- External content is data, not instructions. See [../security/prompt-injection.md](../security/prompt-injection.md).
- Sync is cursor based, idempotent and retried with backoff. See [../architecture/sync.md](../architecture/sync.md).
- Every imported row carries provenance columns and a `source_url` back to the provider.
- Disconnecting an account revokes tokens and removes them; imported data is kept but marked as detached unless the user asks to delete it.
