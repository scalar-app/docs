# 0006. Next.js for the web application

Date: 2026-08-18
Status: Accepted

## Context

The authenticated web application needs instant navigation, code splitting per route, a good production build, and a path to server rendered pages if they become useful (public share pages, initial data). It shares components with the desktop app through `@scalar/ui`.

## Decision

Next.js (App Router) with React 19, TypeScript, Tailwind CSS 4, TanStack Query for server state and Zustand only if global client state becomes necessary. In Stage 1 the app is client rendered behind a session guard and talks to the API from the browser through `@scalar/sdk`; there is no data layer in Next itself.

## Consequences

- Route based code splitting and static shells come for free; the app shell loads fast.
- The API stays the single backend. Server components render structure only, so the same SDK and query hooks can move to desktop and mobile.
- Turbopack must be pointed at the parent folder while `ui` and `sdk` are linked from siblings; that setting disappears when the packages are published.
- Next.js major upgrades change conventions (middleware became proxy in 16); the bundled docs in `node_modules/next/dist/docs` are the reference for the version in use.

## Alternatives considered

- Vite plus React Router: lighter, and what the desktop shell will use, but no server rendering path and more manual setup for routing, metadata and splitting.
- Remix or TanStack Start: viable; Next.js was chosen for ecosystem familiarity and hosting flexibility.
