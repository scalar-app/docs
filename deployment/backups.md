# Backups

Everything Scalar knows lives in PostgreSQL, plus a handful of secrets in your `.env`. Back up those two and you can rebuild the rest.

## What is worth backing up

| What | Where | If you lose it |
| --- | --- | --- |
| The database | `postgres-data` volume | Everything: tasks, events, projects, focus history, integration links |
| `TOKEN_ENCRYPTION_KEY` | `.env` | Integration tokens become undecryptable. Reconnect each account |
| `COOKIE_SECRET` | `.env` | Everyone is signed out. Harmless |
| `INTERNAL_API_TOKEN` | `.env` | Change it in both the API and worker together |
| Redis | `redis-data` volume | Nothing durable. Queued jobs and rate limit counters |
| MinIO | `minio-data` volume | Nothing yet. Attachments are not implemented |

**Keep `.env` with the dump.** A database backup without `TOKEN_ENCRYPTION_KEY` restores your work but not your connected accounts.

## Taking a backup

```bash
docker compose exec -T postgres pg_dump -U scalar scalar | gzip > scalar-$(date +%F).sql.gz
```

Nightly, from cron:

```bash
0 3 * * * cd /srv/scalar/infra && docker compose exec -T postgres pg_dump -U scalar scalar | gzip > /backups/scalar-$(date +\%F).sql.gz
```

Keep the `.env` alongside it, and keep both somewhere that is not the machine running Scalar.

## Restoring

```bash
docker compose up -d postgres
gunzip -c scalar-2026-08-19.sql.gz | docker compose exec -T postgres psql -U scalar scalar
docker compose -f docker-compose.yml -f compose.apps.yml up -d
```

Then open Settings and check Diagnostics: it will tell you whether the database, worker and integrations came back.

## Verifying a backup

A backup nobody has restored is a hope, not a backup. Once, restore into a scratch database and count something:

```bash
createdb -h localhost -p 5433 -U scalar scalar_restore_test
gunzip -c scalar-2026-08-19.sql.gz | psql -h localhost -p 5433 -U scalar scalar_restore_test
psql -h localhost -p 5433 -U scalar scalar_restore_test -c 'select count(*) from tasks'
```

## Getting your data out

Scalar has no export endpoint yet. `pg_dump` is the honest answer today, and the schema is documented in [architecture/data-model.md](../architecture/data-model.md) so a dump is readable without Scalar. Structured export is on the roadmap and the data model was designed to make it straightforward.
