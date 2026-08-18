# OAuth and token handling

Status: planned. There are no integrations in Stage 1, so none of this code exists yet. It is written down now so the first integration is built to it.

## Storage

- Tokens live in `integration_tokens`, one row per integration account: encrypted `access_token`, encrypted `refresh_token`, `expires_at`, `scopes`, `token_type`, `created_at`, `updated_at`.
- Encryption at rest uses an authenticated cipher (AES-256-GCM) with a key from `TOKEN_ENCRYPTION_KEY`, separate from the session secret. Rows store the key version so the key can be rotated.
- Plain tokens are never written to logs, never returned by any API endpoint, and never sent to the web app.

## Credential service

One module in `api` (used by `worker` through the same code) is the only place that decrypts tokens:

- `getAccessToken(accountId)`: returns a valid access token, refreshing first if it expires within a safety margin.
- `storeTokens(accountId, tokens)`: encrypts and upserts.
- `revoke(accountId)`: calls the provider revoke endpoint, then deletes the row.

Provider adapters call the credential service and never read `integration_tokens` directly.

## Rotation

- Access tokens are refreshed on demand using the refresh token. Providers that rotate refresh tokens on refresh get the new one stored immediately.
- Refresh failures with `invalid_grant` mark the account `status = reauth_required`; the user is asked to reconnect. Other failures are retried with backoff.
- The encryption key can be rotated by re-encrypting rows in a background job; the key version column makes this incremental.

## Revocation

When Scalar stops using a token it revokes it at the provider where the provider supports it (Google `oauth2/revoke`; Canvas token deletion for OAuth tokens, or the user deletes a personal access token in Canvas themselves).

## Disconnect flow

1. User clicks Disconnect on the integration settings page.
2. `api` marks the account `disconnecting` and enqueues a job.
3. `worker` calls `revoke` at the provider (best effort, logged on failure).
4. `worker` deletes the `integration_tokens` row.
5. Sync state for the account is set to `disabled`; scheduled jobs are cancelled.
6. Imported data is kept and marked as detached (its `source_account_id` remains for provenance, and the UI shows the source as disconnected). The user can choose to delete imported data as a separate action.
7. `api` marks the account `disconnected`. The user can reconnect later, which creates a new grant.

## Minimum scopes

Each integration declares the minimum scopes for the feature in use. Adding a scope means a new consent screen for the user and a note in the changelog. See [../integrations/README.md](../integrations/README.md).

## Personal access tokens (Canvas self-hosters)

Treated identically to OAuth tokens for storage and use. They cannot be refreshed; when they stop working the account becomes `reauth_required`.
