# Local setup

## Prerequisites

- Node 24. Every repo has an `.nvmrc` containing `24`; run `nvm use` (or `fnm use`) in each.
- pnpm 11. Each `package.json` sets `"packageManager": "pnpm@11.17.0"`, so `corepack enable` is enough.
- Docker with Compose, for Postgres, Redis and MinIO.
- Git.

## Clone the sibling repos

All repositories live as siblings in one parent folder. Nothing works across repos otherwise, because `web` links `sdk` and `ui` by relative path.

```text
scalar/
  .github/
  docs/
  ui/
  sdk/
  api/
  web/
  website/
```

```sh
mkdir scalar && cd scalar
for r in ui sdk integrations api worker web website docs; do git clone git@github.com:scalar-app/$r.git; done
```

## Infrastructure

The docker compose file lives in the `api` repository:

```sh
cd api
docker compose up -d      # PostgreSQL 17, Redis 7, MinIO
```

Host ports are 5433 (PostgreSQL), 6380 (Redis), 9000 and 9001 (MinIO), chosen so other local databases keep working. The compose project is named `scalar`. `api` uses PostgreSQL and Redis (magic link rate limiting); MinIO is started for parity with later stages.

## Environment files

Each repo that reads environment variables ships `.env.example`. Copy it and fill in values:

```sh
cp .env.example .env
```

On Windows that is `copy .env.example .env` in PowerShell, or the same `cp` in Git Bash.

`api` and `worker` load `.env` themselves in development, through Node's built in `--env-file-if-exists`. Nothing loads a file in production: pass real environment variables there, which is what the Dockerfiles and any host expect.

`api`: `DATABASE_URL`, `REDIS_URL`, `APP_ORIGIN` (the web app origin, `http://localhost:3000`), `COOKIE_SECRET` (32+ characters), `PORT` (4000). `web`: `NEXT_PUBLIC_API_URL` (`http://localhost:4000`) in `.env.local`. Each repo's `.env.example` is authoritative. Never commit `.env` files.

## Run order

Build the shared packages first, then the API, then the app.

```sh
cd ui           && pnpm install && pnpm build
cd ../sdk          && pnpm install && pnpm build
cd ../integrations && pnpm install && pnpm build
cd ../api          && pnpm install && pnpm db:migrate && pnpm dev
cd ../worker       && pnpm install && pnpm dev
cd ../web          && pnpm install && pnpm dev
```

- `api` listens on `http://localhost:4000`.
- `worker` needs `REDIS_URL`, `API_URL` and the same `INTERNAL_API_TOKEN` as the API. Without it, integrations connect but never sync.
- `web` runs on `http://localhost:3000` and calls `api` from the browser with cookies. `web/next.config.ts` points Turbopack at the parent folder so the linked `ui` and `sdk` packages resolve.
- Sign in: enter any email on the login page. Without `SMTP_HOST` set, the API returns the verify link and the page shows it, which is the normal way to work locally. With SMTP configured the link is emailed and never returned.

`website` is independent: `cd website && pnpm install && pnpm dev`.

## Rebuilding linked packages

When you change `ui` or `sdk`, run `pnpm build` there again (or `pnpm dev` if the package offers a watch build). `web` picks up the new `dist/`.

## Common scripts

Every package has `dev`, `build`, `lint`, `typecheck`, `test`, `format`. CI runs install, lint, typecheck, test, build.
