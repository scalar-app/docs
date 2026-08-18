# Repositories

Scalar is split into one git repository per concern under the `scalar-app` GitHub organization. Locally, every repository is cloned as a sibling directory under one parent folder. See [ADR 0003](../adr/0003-repo-per-concern-not-monorepo.md) for the reasoning.

## Stage 1 repositories

| Repo | Package | Purpose | Depends on |
| --- | --- | --- | --- |
| `.github` | none | Organization profile, issue templates, shared license text | none |
| `docs` | none | This documentation | none |
| `ui` | `@scalar/ui` | Design tokens (`tokens.css`), React components | none |
| `sdk` | `@scalar/sdk` | Typed TypeScript client for `/api/v1` | none at runtime |
| `api` | private | Fastify HTTP API, Drizzle schema and migrations, docker compose for Postgres, Redis, MinIO | PostgreSQL |
| `web` | private | Next.js web app | `@scalar/sdk`, `@scalar/ui`, `api` |
| `website` | private | Astro public website | copies `tokens.css` from `ui` |

## Planned repositories

| Repo | Purpose |
| --- | --- |
| `worker` | Background jobs: integration sync and the reconciliation scheduler. No database access; talks to the API's internal endpoints |
| `ai` | Provider abstraction, tools, evaluation datasets |
| `mobile` | Mobile client |
| `desktop` | Tauri desktop shell around the web app |
| `infra` | Deployment manifests |
| `integrations` | Provider code (OAuth, sync, normalization). Pure TypeScript, used by both `api` and `worker` |

## Package linking before npm publish

`@scalar/ui` and `@scalar/sdk` are not published to npm yet. `web` depends on them with pnpm's `link:` protocol:

```json
"@scalar/sdk": "link:../sdk",
"@scalar/ui": "link:../ui"
```

Both packages build to `dist/` and expose `exports` with types. You must clone `sdk` and `ui` as siblings of `web` and run `pnpm build` in each before running `web`.

## Shared conventions

Every repository has: `README.md`, `LICENSE` (AGPL-3.0, copyright "Scalar contributors"), `.gitignore`, `.editorconfig`, `.nvmrc` (`24`), `.prettierrc`, `.prettierignore`, `.github/workflows/ci.yml`, and `.env.example` where environment variables are used. See [development/coding-standards.md](../development/coding-standards.md).
