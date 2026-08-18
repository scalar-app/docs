# Authorization

Every request to a resource answers four questions:

| Question | Source |
| --- | --- |
| Who | The user attached to the `scalar_session` cookie |
| What resource | Task, space, event, workspace, integration account |
| Which workspace | The `workspace_id` on the resource, or the workspace the request targets |
| Which permission | read, write, admin, derived from `workspace_members.role` |

A request is allowed only if the user is a member of the resource's workspace with a role that grants the permission. Resources are never looked up by id alone; the query always includes the workspace constraint, so a foreign id returns `NOT_FOUND`, not `FORBIDDEN`.

## Where checks live

Authorization is enforced in the service layer of `api`. Route handlers resolve the session and pass the user to services; services check membership before touching data. The web app hides controls the user cannot use, but that is presentation, not enforcement.

The same services are called by the AI tool runtime ([ai-safety.md](ai-safety.md)) and by the worker, so the same checks apply there.

## Stage 1

- One user, one personal workspace, role `owner`.
- All Stage 1 endpoints operate on the user's workspaces only. `GET /api/v1/workspaces` lists them.
- Roles other than `owner` and shared workspaces are planned. The schema has `role` on `workspace_members` so adding `member` and `viewer` does not need a migration of shape, only of behaviour.

## Authentication vs integration authorization

These are separate concerns and separate tables:

- Authentication is how a person proves they are a Scalar user (magic link today, other providers later, `sessions`).
- Integration authorization is Scalar being allowed to act on an external account on the user's behalf (`integration_accounts`, `integration_tokens`, OAuth grants to Google, Canvas).

Signing in with Google, when it exists, does not grant calendar access, and connecting Google Calendar does not sign anyone in. See [security/oauth-and-tokens.md](../security/oauth-and-tokens.md).
