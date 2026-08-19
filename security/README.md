# Security

## Pages

- [oauth-and-tokens.md](oauth-and-tokens.md): storing, rotating and revoking integration tokens; disconnect flow
- [prompt-injection.md](prompt-injection.md): external content is data, not instructions

## Baseline (Stage 1)

- Sessions: server-side, opaque cookie `scalar_session`, `HttpOnly`, `SameSite=Lax`, `Secure` in production.
- Magic link tokens: stored hashed, single use, expiring. Delivered over SMTP when configured; without it the link is returned in the response, which production refuses unless `ALLOW_LINK_WITHOUT_EMAIL=true`.
- Authorization in the service layer, scoped by workspace membership. See [../architecture/authorization.md](../architecture/authorization.md).
- Input validated with zod at every HTTP boundary.
- Every response carries `x-request-id`; logs are keyed by it.
- No secrets in repositories. `.env.example` has placeholder values only.

## Secrets

- Runtime secrets (database URL, session secret, token encryption key, OAuth client secrets) come from environment variables or the deployment's secret store.
- The token encryption key is separate from the session secret and can be rotated independently.
- Secrets are never logged. Logging is structured, and known secret fields are redacted.

## Reporting a vulnerability

Do not open a public issue. Contact the maintainers privately through the organization's security contact (to be published in the `.github` repository `SECURITY.md`). Until that exists, use the email listed on the organization profile.

## Not yet done

- Rate limiting on auth endpoints.
- CSRF token for state-changing requests beyond `SameSite=Lax` (needed once cross-site clients such as the desktop shell exist).
- Audit logs (`audit_logs` table is planned).
- Security review of AI tool execution (no AI code exists yet).
