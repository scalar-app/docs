# Scalar documentation

This repository holds the documentation for Scalar, an open-source, cross-platform "operating system for attention". Scalar connects email, calendar, school systems (Canvas), tasks, projects and files into one action layer.

Organization: [github.com/scalar-app](https://github.com/scalar-app). License: AGPL-3.0.

## What is in here

Plain Markdown. There is no static site generator yet; one can be added later without changing the file layout.

| Folder | Contents |
| --- | --- |
| `architecture/` | System overview, repository map, data model, sync, AI safety, authorization |
| `api/` | HTTP API reference (`/api/v1`) and conventions |
| `sdk/` | The `@scalar/sdk` TypeScript client |
| `integrations/` | Provider integrations (Google, Canvas) and the integration interface |
| `development/` | Local setup, coding standards, testing |
| `security/` | OAuth token handling, secrets, prompt injection |
| `deployment/` | Self-hosting, AI providers, backups, upgrading, troubleshooting |
| `contributing/` | How to contribute across repos |
| `adr/` | Architecture decision records |
| `roadmap.md` | MVP and later phases |

## Status

Scalar is at Stage 2: Google Calendar sync works end to end, and Scalar Command answers questions from your own data and proposes changes you approve. Many pages still describe planned behaviour. Each page states what is implemented and what is not. When a page says "planned", the code does not exist yet.

There is no hosted Scalar and none is planned. You run it yourself; see [deployment/what-it-costs.md](deployment/what-it-costs.md).

## How the repositories fit together

Scalar is not a monorepo. Each concern is its own repository under the `scalar-app` organization: `.github`, `docs` (this repo), `ui`, `sdk`, `api`, `web`, `website`, `integrations`, `worker`, `ai`, and later `mobile`, `desktop`, `infra`. See [architecture/repositories.md](architecture/repositories.md).

## Contributing docs

1. Write plain Markdown. Keep lines readable; the linter does not enforce a line length.
2. No em dashes. Use commas, periods, colons or parentheses.
3. No marketing language. State facts, and mark anything unimplemented as such.
4. Prefer relative links between pages.
5. Run `npx markdownlint-cli2 "**/*.md"` before opening a pull request. CI runs the same check.
6. Use conventional commits (`docs: ...`).

Architecture decisions go in `adr/` using the template in [adr/README.md](adr/README.md).
