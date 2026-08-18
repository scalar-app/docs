# 0007. Astro for the public website

Date: 2026-08-18
Status: Accepted

## Context

The public site is mostly static content: product explanation, integrations and status, privacy, security, downloads. It must deploy to GitHub Pages through GitHub Actions, score well on Lighthouse, accessibility and SEO, and ship almost no JavaScript.

## Decision

Astro with TypeScript and Tailwind CSS 4, static output. React islands only where a component is genuinely interactive. Design tokens are copied from `scalar-app/ui` (the canonical source) into the site's CSS.

## Consequences

- Zero client JavaScript on the home page today; the Today illustration is plain HTML and CSS.
- The site works at `https://scalar-app.github.io/website/` and at an origin root or custom domain by changing one environment variable; every internal link goes through a base aware helper.
- Tokens are duplicated by copy, so a change in `ui` needs a follow up in `website`. The file is small and the copy is documented.

## Alternatives considered

- Reusing Next.js: heavier build and runtime for a static site.
- Plain HTML: no components, no content collections, harder to keep pages consistent.
