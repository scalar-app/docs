# What it costs

Scalar is a personal project, not a business. Nothing about it is designed to bill anybody, and the project itself is built to cost its maintainers nothing to keep alive.

## There is no Scalar service to pay for

There is no hosted Scalar, and none is planned. There is no account to create with us, no free tier, no upgrade. If you want to use Scalar, you run it. That is the whole distribution model.

This is a deliberate choice rather than a gap waiting to be filled. A hosted service means servers, a database somebody has to back up, an on-call rotation, a support inbox, and a bill that grows with every person who signs up. That turns a project into an obligation. Scalar stays something you can run on a laptop.

The practical consequence for the project: no cloud bill, no domain renewal, no paid CI.

| Thing | Where it runs | Cost |
| --- | --- | --- |
| Source | GitHub public repositories | free |
| CI | GitHub Actions on public repositories | free |
| Website | GitHub Pages at `scalar-app.github.io` | free |
| Docs | this repository | free |
| The app itself | your machine, or a server you already have | yours |
| Model calls | your own API key | yours, and only if you want Ask |

There is no custom domain, because a domain is a recurring bill and `scalar-app.github.io` works.

## What it costs you to run it

Running Scalar for yourself needs Postgres, Redis and Node. All three are free and run happily on a laptop, a spare machine, or the smallest instance any provider sells. `docker compose up -d` covers the databases; see [self-hosting.md](self-hosting.md).

If you already have a machine that stays on, Scalar costs nothing to run.

## Ask, and the only thing that does cost money

Everything in Scalar works without a model key: tasks, calendar, Today, spaces, sync, search over your own data. Ask is the one feature that talks to a language model, and it uses **your** key, billed to **your** Anthropic account.

Set `AI_PROVIDER` on the API to turn it on, pointing at a hosted vendor or at a local model that costs nothing to run. Leave it unset and the API starts normally, `/api/v1/command` returns 503 `AI_UNAVAILABLE`, and the Ask page explains that the feature is not set up on this server. Nothing else is affected. This is why Ask is a page you can ignore rather than a thing woven through every screen.

Two settings keep that bill predictable:

- `AI_DAILY_MESSAGE_LIMIT` (default 100) caps Command messages per person per day. Set it to `0` to remove the cap.
- `AI_MODEL` (default `claude-opus-5`) selects the model. A smaller model costs less per message.

Scalar also spends fewer tokens than a naive assistant would, because the parts that do not need a model do not use one: free time is computed by ordinary calendar arithmetic in `@scalar/ai`, tool results are trimmed before they reach the model, and past tool calls are not replayed into later turns.

## If you contribute

You need nothing paid to work on Scalar. The test suites run against local Postgres and Redis, and the `ai` package tests use a scripted provider rather than a real model, so a full `pnpm test` in any repository needs no API key and makes no network calls.
