# Coding standards

These apply to every Scalar repository.

## Toolchain

- Node 24 (`.nvmrc` = `24`), pnpm 11 (`"packageManager": "pnpm@11.17.0"`).
- TypeScript strict mode with `"noUncheckedIndexedAccess": true`. ESM only (`"type": "module"`).
- ESLint 9 flat config (`eslint.config.js`) with typescript-eslint.
- Prettier: semicolons, single quotes, `printWidth` 100, `trailingComma: all`. `.prettierrc` and `.prettierignore` in each repo.
- `.editorconfig` in each repo.
- Vitest for unit and integration tests. Playwright only where end-to-end tests are required (web, later).

## Package scripts

Every package defines `dev`, `build`, `lint`, `typecheck`, `test`, `format`.

## Required files

`README.md` (what is this repo, how it fits into Scalar, how to run it, dependencies, tests, how to contribute, and a Status section), `LICENSE` (AGPL-3.0, copyright "Scalar contributors"), `.gitignore`, `.editorconfig`, `.nvmrc`, `.prettierrc`, `.prettierignore`, `.github/workflows/ci.yml`, `.env.example` where env is used (never real values).

## CI

`.github/workflows/ci.yml` runs install, lint, typecheck, test, build on pushes to `main` and on pull requests, using `pnpm/action-setup` and `actions/setup-node@v4` with `cache: pnpm`.

## Commits and branches

Conventional commits (`feat:`, `fix:`, `refactor:`, `docs:`, `chore:`, `test:`). Branch prefixes `feat/`, `fix/`, `refactor/`, `docs/`.

## Code rules

- No `any`. Use `unknown` and narrow.
- No placeholder implementations presented as finished. If something is not implemented, say so in the README under Status.
- No filler comments, no TODO spam. A comment explains why, not what.
- Validate at boundaries (HTTP input, env, provider responses) with zod; trust types inside.
- Authorization in the service layer, never only in UI. See [../architecture/authorization.md](../architecture/authorization.md).
- Errors follow the API error shape. See [../api/conventions.md](../api/conventions.md).
- No secrets in code, logs or docs.

## Writing rules (code, docs, UI copy)

- No em dashes. Use commas, periods, colons or parentheses.
- No marketing language in READMEs.
- Short sentences. Plain words.

## Design tokens

Canonical in `ui` (`tokens.css`). Background `#080808`, surface `#101010`, surface-raised `#161616`, border `#242424`, text-primary `#F5F5F3`, text-secondary `#9A9A94`, text-muted `#666662`, scalar-yellow `#FFD600` (used sparingly: focus, selection, active controls, AI activity, branding), yellow-foreground `#080808`, danger `#FF5C5C`, success `#4ADE80`, warning `#FFB020`, info `#7DD3FC`. Radii 4/6/8/12. Fonts: UI system stack starting with Inter; mono JetBrains Mono. Motion 120/200/320 ms with `cubic-bezier(0.2, 0, 0, 1)`; respect `prefers-reduced-motion`. `website` copies `tokens.css` from `ui`; do not edit the copy.
