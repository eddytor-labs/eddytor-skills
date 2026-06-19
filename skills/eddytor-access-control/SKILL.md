---
name: eddytor-access-control
description: >-
  Explains who can do what in Eddytor: the four roles (Admin, Builder, Editor,
  Viewer), the permission matrix, minting and scoping API keys, and registering
  or managing OAuth clients. Activates for roles, permissions, access control,
  "who can do what", API key creation, OAuth client registration, audit log, or
  scope questions.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Eddytor Access Control (roles, API keys, OAuth clients)

Permission in Eddytor is **role-only**. There is no seat type, plan tier, or
subscription gate — what a caller may do is decided entirely by their role
(and, for API-key callers, the scopes minted on the key). This skill covers the
role model, the permission matrix, minting API keys, OAuth client registration,
and reading the audit log.

## Default procedure

1. **Identify the role tier you need.** Four roles, strictly cumulative — each
   inherits every scope of the one below it:
   - **Viewer** — read-only: query/read tables, read domains/connections/storage,
     download objects, read users/org, read API keys, link/unlink own OAuth.
   - **Editor** — Viewer **+** row DML (insert/update/delete), import data,
     create/delete connections, write storage, run AI queries.
   - **Builder** — Editor **+** create/modify/delete tables, write domains,
     delete objects, create/update/revoke API keys, manage AI credentials.
   - **Admin** — Builder **+** all organisation management: update/delete org,
     invite/role/deactivate/delete/recover users, **read the audit trail**,
     manage SSO and provider OAuth apps.
   The exact scope-per-cell mapping is the authoritative matrix in
   `docs/PERMISSIONS.md` — consult it rather than guessing a specific endpoint.

2. **For an API-key caller, remember the intersection rule.** The effective
   permission is **role ∩ key scopes**. A key can never exceed its user's role,
   and narrowing the key's scopes restricts it further within that role.

3. **Mint API keys with `eddytoradm`, inside the server pod/container.**
   `eddytoradm` is the operator CLI; its auth boundary is exec access (the
   `kubeadm` model), so there is no network endpoint or bootstrap token. It has
   exactly **two** subcommands: `setup` and `create-api-key`.

4. **Register/manage OAuth clients via the IdP** (RFC 7591 / 7592) for CLI, MCP,
   or web SPA integrations.

5. **Audit who did what** with the Admin-only `get_audit_log` MCP tool (mirrors
   `GET /v1/organisations/:id/audit` and the CLI `audit log`).

## Tool usage examples

### First admin + organisation (idempotent)

Run inside the server container — no-op once any admin exists:

```bash
# docker compose (self-host single host)
docker compose exec eddytor-server \
  eddytoradm setup --email you@example.com --org "Your Org"

# Kubernetes
kubectl -n eddytor exec deploy/eddytor-server -- \
  eddytoradm setup --email you@example.com --org "Your Org"
```

### Mint an Admin-scoped API key for an existing user

Keys are Admin-scoped, printed **once**, and start with the `edd_live_` prefix.
Omit `--expires-in-minutes` for a non-expiring key.

```bash
# Non-expiring key
kubectl -n eddytor exec deploy/eddytor-server -- \
  eddytoradm create-api-key --email you@example.com

# Labelled key that expires in 7 days (10080 minutes)
kubectl -n eddytor exec deploy/eddytor-server -- \
  eddytoradm create-api-key --email you@example.com \
  --name "ci-pipeline" --expires-in-minutes 10080
```

Use the key as a bearer token: `Authorization: Bearer edd_live_…`.

### Register an OAuth client (RFC 7591 dynamic registration)

```bash
curl -X POST https://eddytor.example.com/register \
  -H 'Content-Type: application/json' \
  -d '{
        "client_name": "my-cli",
        "redirect_uris": ["http://127.0.0.1:8976/callback"],
        "grant_types": ["authorization_code", "refresh_token"]
      }'
```

The response includes `client_id` and a one-time `registration_access_token`.
Per-client expiry comes from `server.oauth_dynamic_client_ttl_secs` (default
30 days; `0` = never).

### Manage a registered client (RFC 7592)

Authenticate with the Bearer `registration_access_token` returned at
registration (it is issued once — store it):

```bash
curl https://eddytor.example.com/register/CLIENT_ID \
  -H "Authorization: Bearer REGISTRATION_ACCESS_TOKEN"          # GET (read)
# PUT to update, DELETE to deregister — same Bearer token.
```

### Read the organisation audit log (Admin-only)

MCP tool — returns the org activity log, resolving each acting user's email so
admins can see who performed each action:

```json
{ "tool": "get_audit_log", "arguments": {} }
```

## Gotchas

- **Permission is role-only.** Do not look for a plan/tier/seat toggle — none
  exists. If a caller lacks a capability, change their **role** (or widen the
  key's scopes within that role).
- **API-key scopes only narrow, never widen.** Effective = role ∩ key scopes; a
  key on a Viewer can never gain Editor scopes.
- **`eddytoradm` keys are always Admin-scoped** and tied to an **existing** user
  (`mint_admin_api_key`). To issue a lower-privilege key, use the API-key REST
  flow (`apikeys:create`, a Builder/Admin scope), not `eddytoradm`.
- **`eddytoradm` runs only inside the server pod/container** — it shares the
  server's DB connection and secrets and exposes no network endpoint. Minting a
  key requires `EDDYTOR_API_KEY_SECRET` to be set.
- **Only two `eddytoradm` subcommands exist** — `setup` and `create-api-key`.
  There is no user-delete, role-change, or list command in `eddytoradm`; user
  management is the Admin REST/CLI/MCP surface.
- **The key is printed once.** There is no way to retrieve it later — capture it
  immediately. Rotating `EDDYTOR_API_KEY_SECRET` invalidates **all** keys.
- **`registration_access_token` is issued once.** Lose it and you cannot
  GET/PUT/DELETE that client registration.
- **Audit reads are Admin-only** across REST, CLI, and the `get_audit_log` MCP
  tool — a Builder cannot read it.

## Guidelines

- Grant the **lowest role** that does the job: Viewer for read-only consumers,
  Editor for data entry, Builder for schema authors, Admin only for operators.
- Prefer **short-lived, labelled** API keys (`--name`, `--expires-in-minutes`)
  for automation so they can be reasoned about and rotated.
- Treat `docs/PERMISSIONS.md` as the source of truth for exact scope-to-endpoint
  mappings; scope definitions live in
  `crates/common/src/http/middleware/permission/scope_definitions.rs`.
- Use OAuth dynamic registration for first-party clients (CLI/MCP/web SPA); use
  `eddytoradm` API keys for headless/server-to-server access.
