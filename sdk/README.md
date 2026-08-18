# @scalar/sdk

Typed TypeScript client for the Scalar API. Lives in the `sdk` repository. It mirrors the contract in [../api/conventions.md](../api/conventions.md); the API repository owns the contract, the SDK follows it.

## Status

Not published to npm. Consumers use pnpm `link:`:

```json
"@scalar/sdk": "link:../sdk"
```

Clone `sdk` as a sibling of the consuming repo and run `pnpm build` in it first. It builds to `dist/` with type declarations and an `exports` map. ESM only.

## What it provides

- Types for every Stage 1 resource: `User`, `Workspace`, `Space`, `Task`, `Event`, `TodayResponse`, `TaskStatus`, `TaskPriority`, `Paginated<T>`, `ApiError`.
- A client with one method per endpoint, taking typed params and returning typed results.
- Error handling: non-2xx responses throw an error carrying `code`, `message`, `status` and `requestId` from the `{ error }` body and `x-request-id` header.
- Cookie-based auth: the client sends credentials with every request. It does not manage tokens.

Illustrative usage; check the `sdk` README for the exact exported names:

```ts
import { createClient } from '@scalar/sdk';

const scalar = createClient({ baseUrl: 'http://localhost:3000' });

const { data, nextCursor } = await scalar.tasks.list({ status: ['todo', 'in_progress'] });
const today = await scalar.today.get({ date: '2026-08-18', tz: 'Europe/Berlin' });
```

## Rules

- No `any`. Strict TypeScript, `noUncheckedIndexedAccess`.
- No dependency on React or on `@scalar/ui`.
- Uses `fetch`; a custom `fetch` can be injected for tests and non-browser runtimes.
- Pagination helpers iterate `nextCursor` until `null`.

## Versioning

Until publication, the SDK tracks `api` main. After publication it will follow semver, with breaking API changes producing a major bump.
