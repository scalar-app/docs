# Self-hosting

Scalar is AGPL-3.0 and designed to be run by anyone. There is no hosted Scalar and none is planned: running it yourself is the only way to use it, and the only way it is meant to be used. See [what-it-costs.md](what-it-costs.md).

## Components

| Component | Role | Stage 1 |
| --- | --- | --- |
| `api` | HTTP API, migrations, sessions | required |
| `web` | Next.js app served to browsers | required for the UI |
| `worker` | Background jobs, calendar sync, dead-letter | required for sync |
| PostgreSQL | Primary datastore | required |
| Redis | Queue, cache, rate limits | required |
| S3-compatible storage (MinIO locally) | Files and attachments | started by compose; unused by Stage 1 `api` |
| `website` | Public marketing site | optional, static |

## Minimal deployment (Stage 1)

1. Provision PostgreSQL 16 or newer.
2. Deploy `api` (Node 24, or the provided Dockerfile which runs migrations on start) with `DATABASE_URL`, `REDIS_URL`, `APP_ORIGIN`, `COOKIE_SECRET`, `PORT`, `NODE_ENV=production`.
3. Deploy `web` (Next.js) with the public API URL. Serve `api` and `web` on the same site (same registrable domain) so the `SameSite=Lax` session cookie works; put both behind one reverse proxy with TLS.
4. Set `Secure` cookies by running with `NODE_ENV=production` behind HTTPS.

Magic link email is not implemented. A self-hosted Stage 1 deployment can only sign in by reading the verify link from the `api` logs. This is acceptable for personal use and not for anything else; email delivery is on the roadmap.

## Later stages

- `worker` runs alongside `api`, sharing `DATABASE_URL` and a Redis URL.
- Integrations need your own OAuth clients (Google Cloud project, Canvas developer key or personal tokens) configured via environment variables.
- `TOKEN_ENCRYPTION_KEY` for integration tokens; back it up, since losing it means every user must reconnect.
- `ANTHROPIC_API_KEY` if you want Ask. Without it the API starts normally and Command returns 503; nothing else changes. The key is yours and is billed to your own account. `AI_DAILY_MESSAGE_LIMIT` caps messages per person per day.
- S3-compatible storage for attachments.

## Reference files

- `api/docker-compose.yml`: local Postgres, Redis, MinIO.
- Deployment manifests are planned for an `infra` repository. None exist yet.

## Backups

Back up PostgreSQL. Everything Scalar knows is there. Object storage, once used, must be backed up too.

## Upgrades

Pull new versions of `api` and `web`, run migrations, restart. Migrations are forward-only.
