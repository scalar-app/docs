# Testing

## Tools

- Vitest everywhere for unit and integration tests.
- Playwright only for end-to-end tests where they are required (web, later). Not set up in Stage 1.
- `pnpm test` in every package. CI runs it after lint and typecheck.

## What to test, per repo

| Repo | Focus |
| --- | --- |
| `api` | Service-layer logic with a real PostgreSQL (docker compose): authorization scoping, task filters, Today computation, magic link token hashing and expiry, error shapes, pagination cursors |
| `sdk` | Request building and response parsing against a mocked `fetch`; error mapping from `{ error }` bodies; pagination iteration |
| `ui` | Component rendering and accessibility basics; token file is exercised by a snapshot |
| `web` | Component tests where logic exists; end-to-end later |
| `website` | Build succeeds; links resolve |
| `docs` | markdownlint |

## Rules

- Tests run against real Postgres for `api`, not an in-memory substitute. Behaviour like `ILIKE` and `timestamptz` must be exercised for real.
- Deterministic. Today computation takes the clock and time zone as inputs so tests can pin them.
- Isolated. Each test creates its own user and workspace; no shared fixtures that leak between tests.
- Fast enough to run on every push. Slow suites are split, not skipped.
- A bug fix comes with a test that fails before and passes after.

## Planned

- Contract tests that run `sdk` against a live `api` to catch drift.
- Evaluation datasets for the `ai` repository; see [../architecture/ai-safety.md](../architecture/ai-safety.md).
- Provider adapter tests using recorded fixtures for Google and Canvas.
