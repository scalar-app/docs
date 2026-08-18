# 0003. One repository per concern, not a monorepo

Date: 2026-08-18
Status: Accepted

## Context

Scalar spans an API, a worker, integrations, an AI layer, an SDK, a design system, three clients, a website, docs and infrastructure. The project is open source and wants contributors who care about one area (a Canvas integration, the mobile app) to clone and run only that area. Each area has its own release cadence and toolchain (Rust for desktop, Expo for mobile, Astro for the website).

## Decision

Each concern is its own repository under the `scalar-app` GitHub organization. Shared code lives in published packages (`@scalar/ui`, `@scalar/sdk`). Organization wide standards, templates and reusable workflows live in `scalar-app/.github`. Locally, contributors clone the repositories they need side by side; `web` consumes `ui` and `sdk` with pnpm's `link:` protocol until they are published.

## Consequences

- Clear ownership and small clones. CI per repository stays fast.
- Cross repository changes (an API change needs SDK, docs and web updates) require discipline: the definition of done in the contributing guide lists the repositories a change must touch.
- Local setup needs a documented run order (build `ui` and `sdk` before `web`). `docs/development/local-setup.md` and each README carry it. A bootstrap tool (`scalar-app/dev`) can be added if this becomes cumbersome.
- Publishing `ui` and `sdk` to npm is required before external consumers can use them.

## Alternatives considered

- Monorepo with pnpm workspaces: simplest local development and atomic cross cutting changes, but a heavy clone for single area contributors, mixed toolchains in one tree, and a single CI surface that grows without bound. Explicitly rejected by project leadership.
