# Contributing

Scalar is developed in the open at [github.com/scalar-app](https://github.com/scalar-app) under AGPL-3.0. Contributions to any repository are welcome.

## Where things live

Pick the repository that owns the concern; see [../architecture/repositories.md](../architecture/repositories.md). Cross-repo changes (for example an API field that also needs SDK types and a web change) are three pull requests, opened in dependency order: `api`, then `sdk`, then `web`. Link them to each other.

## Workflow

1. Open or find an issue. Small fixes do not need one.
2. Branch from `main` with a prefix: `feat/`, `fix/`, `refactor/`, `docs/`.
3. Follow [../development/coding-standards.md](../development/coding-standards.md). Run `pnpm lint`, `pnpm typecheck`, `pnpm test`, `pnpm build` locally.
4. Use conventional commits.
5. Open a pull request. Describe what changed and why. Mention anything not finished; do not present it as done.
6. CI must pass. A maintainer reviews and merges.

## Documentation

If a change alters behaviour documented here, update the docs in the same effort (a PR to `docs`). API changes update `api/` pages; schema changes update `architecture/data-model.md`. Decisions with lasting consequences get an ADR ([../adr/README.md](../adr/README.md)).

## Style of writing

Plain, short, honest. No em dashes. No marketing.

## Licensing of contributions

By contributing you agree that your contribution is licensed under AGPL-3.0, the same as the project. There is no CLA.

## Code of conduct

Be respectful. Assume good intent. Maintainers may remove content or contributors who do not.
