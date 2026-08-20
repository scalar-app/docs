# Troubleshooting

Start at **Settings, then Diagnostics**. It reports each component with a sentence rather than a status code, and most of what follows is what those sentences mean.

## Nobody can sign in

**"This server cannot send sign in emails yet."** No `SMTP_HOST` is set and the server is in production. Either configure SMTP ([self-hosting.md](self-hosting.md)), or, if the server is genuinely private, set `ALLOW_LINK_WITHOUT_EMAIL=true` and the API returns the link in the response instead. Be sure about "private": that setting lets anyone who can reach the API sign in as anyone.

**"The sign in email could not be sent."** SMTP is configured but the relay refused. The API logs carry the reason. Common causes: wrong port (587 for STARTTLS, 465 for implicit TLS), a `SMTP_FROM` the relay will not send for, or a password that is not an app password.

**Signed in, then immediately signed out.** The session cookie is marked Secure in production, so it needs HTTPS. Either put TLS in front of Scalar or run with `NODE_ENV=development` on a machine only you use.

## Nothing is syncing

**"The worker has not checked in."** The worker container is not running, or cannot reach the API. `docker compose logs worker`. It needs `API_URL` and the same `INTERNAL_API_TOKEN` as the API.

**"Google Calendar needs reconnecting."** The grant was revoked or expired. Settings, then Integrations, then Reconnect. Imported events stay.

**"Canvas is not syncing."** The message carries Canvas's own reason. A token that was deleted or expired shows as unauthorized; reconnect with a new one.

## Ask Scalar is unavailable

**503 `AI_UNAVAILABLE`.** No provider configured, or the configuration was rejected at boot. The API logs say which variable is missing. See [ai-providers.md](ai-providers.md). Everything except Ask keeps working. That is deliberate, not a partial outage.

**"Could not reach the model provider."** For a local model, the server is not running or `AI_BASE_URL` is wrong. From inside compose, a sibling container is `http://ollama:11434`, not `localhost`.

## The API will not start

**"Invalid environment: ..."** It names the variables. `COOKIE_SECRET` needs 32 characters, `INTERNAL_API_TOKEN` needs 32, `TOKEN_ENCRYPTION_KEY` needs 32 random bytes base64.

**It cannot reach PostgreSQL.** From inside compose the host is `postgres`, not `localhost`. From outside, the port is 5433 rather than 5432, deliberately, so Scalar runs beside a PostgreSQL you already have.

## Integrations decrypt as garbage

`TOKEN_ENCRYPTION_KEY` changed. There is no recovery: disconnect each account and connect it again. This is why that key belongs in your backup.

## Getting more detail

```bash
docker compose logs -f api
docker compose logs -f worker
```

`LOG_LEVEL=debug` on the API for more. Logs redact cookies, tokens and passwords, so they are safe to read and to paste when asking for help. Check anyway before pasting.
