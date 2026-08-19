# Upgrading

```bash
cd infra
git pull
sh scripts/clone.sh          # picks up any new repository
docker compose -f docker-compose.yml -f compose.apps.yml up -d --build
```

The API applies pending migrations on boot. **Take a backup first** ([backups.md](backups.md)) — that is the whole rollback plan, and it takes a minute.

## Check it worked

Settings, then Diagnostics. It reports the running version, how many migrations have been applied, and whether the database, worker, mail, AI provider and integrations are each happy. A worker that has not checked in is the most common thing to notice here.

## Rules the migrations follow

- **Additive.** No table is dropped and no column is repurposed in place. A column being replaced is deprecated and left alone for at least one release; `tasks.source_id` is the current example.
- **Tested against real data.** `api/test/integration/migrations.test.ts` builds a database at the previous version, fills it with rows, migrates, and asserts they survived. That test is why a migration reaching you has been run against something other than an empty schema.
- **Applied on boot**, in order, once. Interrupting the API mid-upgrade leaves the applied ones applied; starting it again continues.

## Deploying the parts together

The API and the worker share an internal contract that is not versioned. **Upgrade them together.** In practice `docker compose up -d --build` does, because it rebuilds both.

The web app and the SDK are versioned by the API's `/api/v1` prefix, so a slightly old web build keeps working.

## Downgrading

There is no down migration. To go back: restore the backup taken before the upgrade, and check out the previous tags. This is the reason the backup step is not optional.

## Breaking changes

| Version | Change |
| --- | --- |
| Unreleased | The worker-to-API sync payload now discriminates on `kind`. Deploy the API and worker together |
| Unreleased | `ANTHROPIC_API_KEY` is deprecated in favour of `AI_PROVIDER` and `AI_API_KEY`. The old variable still works |
