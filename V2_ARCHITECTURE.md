# Scalar V2 architecture audit

Phase 0 deliverable. This is an audit of what exists today plus the proposed shape of V2 and the
order in which to build it. No application code has been changed to produce it.

Audited: local checkouts of every repository in the `scalar` container directory, at the state on
disk on 2026-08-19.

---

## 1. Current architecture map

```text
                 browser                     desktop (Tauri 2)
                    |                                |
                    +--------------+-----------------+
                                   |
                            @scalar/web (Next 16)
                                   |
                            @scalar/sdk (typed client, zod)
                                   |
                          @scalar/api (Fastify 5)
                       modules: auth spaces tasks events
                       today search integrations command
                        |            |              |
                   PostgreSQL      Redis        @scalar/ai
                   (drizzle)      (BullMQ)     @scalar/integrations
                                     |
                              @scalar/worker
                            (BullMQ consumer +
                             reconcile scheduler)
                                     |
                            internal HTTP back to API
```

Two facts about this diagram matter more than the boxes.

The worker holds no database connection. It consumes a sync job, calls the provider, and pushes the
result back through internal API routes guarded by `INTERNAL_API_TOKEN`. Persistence and
authorization stay in one process. That is a good decision and V2 should keep it.

Everything the browser does goes through the SDK. There are no ad hoc `fetch` calls scattered across
components; `web/src/lib/queries/*` wraps SDK calls in TanStack Query hooks.

## 2. Repository responsibilities

| Repo | Package | Role | State |
| --- | --- | --- | --- |
| `api` | `@scalar/api` | Fastify server, Postgres via drizzle, owns all writes and authorization | Solid, actively the centre of gravity |
| `sdk` | `@scalar/sdk` | Typed client, zod response schemas, cursor pagination | Solid, mirrors API surface |
| `ai` | `@scalar/ai` | Model provider abstraction, tool registry, command loop, deterministic scheduling maths | Solid foundation, one provider implemented |
| `integrations` | `@scalar/integrations` | Provider framework and Google Calendar implementation | Works, but no shared integration interface yet |
| `worker` | `@scalar/worker` | BullMQ consumer and reconciliation scheduler | Small and correct |
| `ui` | `@scalar/ui` | Design tokens plus about twenty primitives | Good tokens, thin component set |
| `web` | `@scalar/web` | Next 16 app, App Router, TanStack Query | Covers Stage 2 surface |
| `desktop` | `@scalar/desktop` | Tauri 2 shell over a static export of `web` | Shell only |
| `website` | `@scalar/website` | Astro marketing site | Independent |
| `infra` | n/a | Compose files for datastores and for apps built from source | Working, some rough edges |
| `docs` | n/a | ADRs, API reference, architecture, deployment, security | Unusually complete for this stage |

Repo per concern is recorded in `docs/adr/0003-repo-per-concern-not-monorepo.md`. Local development
links siblings with `link:../ai` style dependencies, and the API and worker Dockerfiles take the
parent directory as build context for the same reason. V2 should not fight this, but it is the main
source of friction when a change spans packages.

## 3. Database and schema

Three migrations exist: `0000_initial`, `0001_integrations`, `0002_ai`. Drizzle generates them, and
a snapshot per migration lives under `src/db/migrations/meta`.

Tables today:

```text
users                     email, name
sessions                  hashed token, expiry
magic_link_tokens         hashed token, single use
workspaces                tenant: personal or team, owner
workspace_members         role: owner, admin, member
spaces                    context grouping inside a workspace
tasks                     status, priority, dueAt, scheduledStart/End,
                          estimatedMinutes, parentTaskId, sourceId
events                    startsAt/endsAt, allDay, provenance columns
integration_accounts      provider, externalAccountId, status, settings
integration_tokens        AES-256-GCM ciphertext, one row per account
integration_sync_state    per resource cursor, status, lastError, nextSyncAt
ai_threads / ai_messages / ai_actions
```

Three observations that shape V2.

**The naming collision is the single most important finding.** The V2 brief asks to rename Spaces to
Workspaces. `workspaces` already exists as the tenant and membership boundary; every table carries
`workspace_id`, every service method takes `workspaceId`, and the SDK and API docs use the same word.
Renaming Spaces to Workspaces would overload a term that already means something else in the schema,
the authorization layer and the public API. Section 13 proposes the alternative.

**Tasks already carry most of the planner columns.** `scheduledStart`, `scheduledEnd`,
`estimatedMinutes` and `dueAt` exist and are unused by any scheduling engine. V2 does not need a new
task model, it needs something that fills those columns.

**Provenance is already modelled properly on events** (`source`, `integrationAccountId`,
`sourceObjectId`, `sourceUpdatedAt`, `lastSyncedAt`, with a unique index on account plus source
object). Tasks only have a loose `sourceId` text column. That asymmetry needs fixing before Canvas or
Gmail land.

## 4. API architecture

Fastify 5 with `fastify-type-provider-zod`. Each module is four files: `dto.ts` (zod schemas),
`repository.ts` (drizzle queries), `service.ts` (domain logic), `routes.ts` (wiring). `buildApp`
constructs services explicitly and injects them, which is why the integration tests can run the whole
app against a real database with a fake Google and a scripted model provider.

Cross cutting pieces are in place: typed `AppError` hierarchy, error handler plugin, request context
with request ids, cursor pagination helper, log redaction of cookies and tokens, helmet, CORS pinned
to `APP_ORIGIN`, signed cookies.

Route groups today: `/health`, `/auth`, `/me`, `/spaces`, `/tasks`, `/events`, `/today`, `/search`,
`/integrations`, `/internal/*`, `/command`.

Gaps relative to V2: no `/timeline`, `/planner`, `/focus`, `/projects`, `/inbox`, `/settings`,
`/notifications`. There is no per user settings or preferences storage at all, which the planner
needs (working hours, timezone, buffers).

## 5. SDK

`sdk/src/schemas/*` mirrors the API DTOs one file per domain. The client is a thin typed wrapper over
`http.ts` with typed errors and a cursor pagination helper. Adding a V2 domain means one schema file
and one client namespace. No structural change needed, only additions.

## 6. Frontend

Next 16 App Router, React 19, Tailwind 4, `@scalar/ui` for primitives, TanStack Query for server
state, an `(app)` route group with `AppShell`, `Sidebar`, `MobileNav`, `CommandPalette` and go-to
navigation hotkeys. Views: Today, Ask, Inbox, Tasks, Calendar, Spaces, Search, Settings and
Integrations.

The separation the brief asks for mostly holds already: components consume hooks from
`lib/queries/*`, which consume the SDK. Approval cards for AI actions exist
(`components/command/ApprovalCard.tsx`).

Inbox is deliberately not a table: it is tasks with `status = 'inbox'`, documented in a comment at
the top of `InboxView.tsx`. This is the right instinct and V2 should extend rather than replace it.

Missing for V2: Timeline, Focus, Projects, Plan preview, Needs Attention, Up Next, per feature empty
and error states in some views, and a settings surface beyond integrations.

## 7. Worker

BullMQ `Worker` on `scalar.sync`, concurrency from env, five attempts with exponential backoff, plus
a polling scheduler that asks the API which resources are due for reconciliation. Job ids are derived
from account and resource so a queued sync is not duplicated. Failure handling distinguishes
reauthorization, transient and permanent provider errors.

It is a single queue with a single job type. Everything V2 wants from background work (notifications,
recurring rollover, index maintenance, planner jobs) will be new job types on the same worker, which
is the right answer for self hosting.

## 8. Integration architecture

`@scalar/integrations` currently exports Google OAuth helpers, a Google Calendar normalizer and an
incremental sync using `syncToken`, an HTTP helper, and a typed error taxonomy
(`ReauthorizationRequiredError`, `ProviderTransientError`, `ProviderError`).

What it does not have is the `ScalarIntegration` contract from the brief. `Provider` is a union with
exactly one member, and the sync path in the worker is Google Calendar specific. Nothing is wrong
with the code, it just has not been generalized yet, and it should be generalized against a second
real provider rather than in the abstract.

## 9. AI architecture

This is the strongest part of the codebase relative to V2 and needs the least work.

- `ScalarModelProvider` already abstracts generate, structured output and optional embeddings.
- `AnthropicProvider` and a `scripted` provider for tests implement it.
- The tool registry classifies every tool `read`, `suggest` or `write`, and requires a
  `describeAction` for anything that is not `read`. Classification, not model intent, decides whether
  a call executes or becomes a proposal.
- Proposals persist to `ai_actions` as `pending` and only execute after a person approves.
- `ai/src/scheduling.ts` is deterministic free slot computation with correct timezone and DST
  handling. The model asks for free time; this computes it.
- The API returns 503 from `/command` when no key is configured, and everything else keeps working.

The one real gap is configuration, not design: `env.ts` knows only `ANTHROPIC_API_KEY` and
`AI_MODEL`. There is no `AI_PROVIDER`, no base URL, so an OpenAI-compatible endpoint or Ollama cannot
be pointed at without code changes even though the interface supports it.

## 10. Authentication

Magic link only. Tokens are hashed at rest, single use, rate limited, with a signed
`scalar_session` cookie and a server side `sessions` table. The auth hook resolves user and workspace
on every request; `requireAuth` and `authed()` narrow the request type.

Two open items. Email delivery is not implemented, so the sign in link is returned by the API in
development mode, and `compose.apps.yml` defaults the API to `NODE_ENV=development` because of it.
That is a self hosting blocker and should be treated as one. There is also no password or OIDC path,
which is fine for now.

## 11. Self-hosting today

`infra/docker-compose.yml` brings up PostgreSQL 17, Redis 7 and MinIO. `compose.apps.yml` builds API
and worker from sibling checkouts. Health checks and named volumes are present. `.env.example` files
exist in `api`, `worker`, `web`, `infra` and `website`.

Three friction points:

1. Getting to a running Scalar needs `scripts/clone` plus two compose files, not one command.
2. MinIO runs but nothing uses object storage yet. It is a container self hosters pay for and get
   nothing from today.
3. Redis is mandatory because BullMQ is mandatory. Defensible for now given the sync workload, but it
   is the one infrastructure dependency worth revisiting if Postgres-backed queueing would do.

No Scalar-operated service is required anywhere in the stack. OAuth credentials are the self
hoster's, tokens are encrypted with the self hoster's key, and the AI key is theirs. The
non-negotiable requirement in the brief is currently met, and the audit found nothing that violates
it.

## 12. Technical debt relevant to V2

1. Spaces and Workspaces terminology collision (section 3).
2. Tasks lack the provenance columns events already have; needed before Gmail and Canvas.
3. No user settings or preferences table; the planner cannot exist without one.
4. `Provider` union with one member and a Google-specific sync path in the worker.
5. AI provider configuration is Anthropic-shaped even though the interface is not.
6. Migrations have no down path and no migration tests.
7. Email delivery missing, which forces a development-mode default in the app compose file.
8. `ui` has primitives but no product-level components, so `web` grows its own.
9. Search is `ILIKE` style querying, with no full text index yet.
10. MinIO in compose with no consumer.

None of these is a rewrite. All are incremental.

## 13. Proposed V2 domain model

Keep `workspaces` as the tenant. Rename nothing in the authorization boundary.

Promote **Spaces** to the user-facing context concept the brief calls Workspaces, and add
**Projects** beneath them:

```text
Workspace (tenant, unchanged)
└── Space            School, Personal, Scalar
    └── Project      CSE 13S, V2, Website
        └── Task
```

This gives the two-level hierarchy the brief wants, costs one new table and one nullable column on
tasks, and breaks no existing API, SDK or authorization code. The UI can present Spaces however
reads best; the label is a presentation decision, the schema is not.

New tables proposed for V2, in dependency order:

```text
user_preferences     timezone, working hours, buffers, week start,
                     auto-schedule behaviour, learning opt-out
projects             spaceId, name, status, startAt, dueAt
timeline_blocks      itemId, itemType, startAt, endAt, blockType,
                     locked, source
focus_sessions       taskId, startedAt, endedAt, planned/actual minutes
inbox_items          only if a source needs to hold an unconverted item;
                     tasks with status inbox stay the default
activity             actor, verb, subject, payload, for explanation and undo
```

Task provenance columns to match events: `source`, `integrationAccountId`, `sourceObjectId`,
`sourceUrl`, `sourceUpdatedAt`, `lastSyncedAt`, with the same unique index shape.

## 14. Proposed Timeline architecture

Timeline is a read model over events plus scheduled tasks plus focus sessions, with
`timeline_blocks` as the only new writable surface. Calendar events stay fixed and are never moved by
Scalar; task blocks are movable; blocks may be `locked` to opt out of planning.

`GET /timeline?date=&tz=` returns an ordered sequence of blocks for a day, built the same way
`today/compute.ts` builds Today: repositories fetch, a pure function composes. Reuse
`resolveDayWindow` rather than writing a second timezone implementation.

## 15. Proposed Planner architecture

A pure function in `@scalar/ai`, or better a new `planner` module inside it, with no database,
network or model access:

```ts
plan(request: PlanningRequest): PlanningResult
```

It consumes fixed events, schedulable tasks and preferences, and returns proposed blocks,
unscheduled items, conflicts and warnings, each carrying machine readable reasons
(`due_within_24_hours`, `fits_available_window`, `preferred_focus_period`). `findFreeSlots` in
`ai/src/scheduling.ts` is the availability half of this and should be the foundation.

The API exposes `POST /planner/preview` and `POST /planner/apply`. Apply runs in one transaction so a
plan never half lands. The existing `ai_actions` approval flow is the right precedent, and the
`ApprovalCard` component already exists in `web`.

## 16. Proposed Inbox architecture

Keep the current model: an unfiled task is a task with `status = 'inbox'`. Add a suggestion envelope
rather than a second table, so a Canvas assignment or a Gmail message arriving in the inbox can carry
proposed space, project, duration and schedule that a person accepts, edits or dismisses. Suggestions
are advisory metadata; nothing is applied without a decision.

## 17. Proposed Command architecture

The typed command boundary the brief asks for is most of the way there: the tool registry is that
boundary. V2 work is to add the planner-shaped commands (`schedule_task`, `move_block`,
`reschedule_unfinished`) as `suggest` tools that return planner previews, never as direct writes, and
to keep the rule that AI interprets while the planner decides.

## 18. Proposed AI provider architecture

No interface change. Configuration change only:

```env
AI_PROVIDER=anthropic | openai_compatible | ollama
AI_MODEL=
AI_BASE_URL=
AI_API_KEY=
```

Add an `OpenAICompatibleProvider` that covers both OpenAI and Ollama's compatible endpoint. Keep
`ANTHROPIC_API_KEY` working as a deprecated alias for one release. Degrade gracefully: local models
handle interpretation worse, and every AI-dependent surface must already have a non-AI path, which
today it does.

## 19. V1 to V2 migration strategy

- Additive migrations only. No table is dropped and no column is repurposed in place.
- Spaces are not renamed in the schema. Projects are added beneath them, nullable on tasks.
- Every migration that touches existing rows gets a test that loads V1-shaped data and asserts it
  survives.
- Preferences get server-side defaults so existing users are valid the moment the column exists.
- `sourceId` on tasks is backfilled into the new provenance columns, then left in place until a later
  release removes it.

## 20. Implementation phases

Ordered so each phase is shippable and nothing depends on a later one.

1. **Foundation.** `user_preferences`, `projects`, task provenance columns, migration tests, SDK and
   API additions. UI unchanged except a settings page for working hours and timezone.
2. **Timeline.** Read model, `/timeline`, SDK, Timeline component, Today rebuilt on it.
3. **Planner core.** Pure `plan()` with an extensive test suite covering the cases in the brief.
   No API surface yet.
4. **Planner API and preview.** `/planner/preview` and `/planner/apply`, transactional apply, plan
   preview UI on top of the existing approval pattern.
5. **Focus.** `focus_sessions`, start and complete flows, duration recording, opt-out preference.
6. **Home.** Up Next and Needs Attention as deterministic engines with exposed reasons.
7. **Inbox suggestions.** Suggestion envelope, accept, edit, dismiss.
8. **AI provider config.** `AI_PROVIDER`, OpenAI-compatible provider, docs.
9. **Integration contract.** Generalize against Canvas or Gmail as the second provider.
10. **Self-hosting polish.** One-command deploy, email delivery, health and diagnostics page, backup
    and upgrade docs.

Phases 1 through 4 are the spine. Everything after is additive.

## 21. Risks

- **Terminology.** If Spaces is renamed to Workspaces anywhere in the schema or API, authorization
  code becomes ambiguous. Section 13 avoids this; it is worth an explicit decision before Phase 1.
- **Planner scope.** The temptation to optimize. The brief's heuristic ordering is enough for V2 and
  it stays explainable.
- **Cross-repo changes.** A single domain addition touches `api`, `sdk`, `web` and sometimes `ai`.
  Expect four coordinated commits per phase and keep them small.
- **Email delivery.** Until it exists, secure self-hosting has no sign in path. This is a Phase 10
  item by the ordering above but is arguably urgent enough to pull forward.
- **Timezone duplication.** Timezone arithmetic exists in both `api/src/modules/today/compute.ts` and
  `ai/src/scheduling.ts`. Adding a third copy in the planner would be the beginning of a bug.
  Consolidate before Phase 3.

## 22. Open technical questions

1. Is Spaces plus Projects acceptable, or is a schema-level rename of the tenant concept wanted
   instead? This blocks Phase 1.
2. Should scheduled task blocks ever be written to the external calendar, and if so opt in per space
   or globally?
3. Should Redis stay mandatory, or is a Postgres-backed queue worth the work to drop a container?
4. Multi-user: `workspace_members` and roles exist but nothing enforces sharing semantics yet. Does
   V2 need them, or do they stay dormant?
5. MinIO: keep for planned attachments, or remove from compose until something uses it?
6. Does duration learning default on or off?

---

## Verification notes

Claims here were checked against the source on disk, not inferred from documentation. Test coverage
observed: 34 test files, including API integration tests that boot the whole app against a real
database with a fake Google provider and a scripted model provider. No code was modified.
