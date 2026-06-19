---
name: eddytor-rest-api
description: >-
  Best practices for calling the Eddytor REST API from apps and integrations —
  auth, the OpenAPI spec, the unified error envelope, rate limits, and the
  read/write boundary. Activates for REST API, /api/v1, HTTP integration, bearer
  token, API key, OpenAPI, or webhooks/automation — even without "REST."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Eddytor REST API best practices

The REST API (`https://<host>/api/v1/…`) is the control plane for apps and integrations:
tables, schema, domains, storage, org, and auth. The machine-readable contract is the
**OpenAPI spec served at `/spec`** — generate clients/SDKs from it rather than hardcoding URLs.

## Auth

Send `Authorization: Bearer <credential>` with either:

* an **OAuth 2.1 access token** (short-lived, ~15 min — refresh it), or
* an **API key** (`edd_live_…`) for headless/server-to-server use (mint via
  `eddytoradm create-api-key`; effective permission = role ∩ key scope — see
  eddytor-access-control).

Browser SPAs use the cookie session + **CSRF** token and are subject to the CORS allowlist;
server-side callers should use a bearer token/API key and can ignore CSRF/CORS.

## The unified error envelope

Every error is the same shape — branch on `code`, show `message`, log `request_id`:

```json
{ "code": "not_found", "message": "Table not found",
  "request_id": "550e8400-…", "details": [{ "field": "name", "message": "must not be empty" }] }
```

Codes: `validation_error`, `authentication_error`, `forbidden`, `not_found`, `conflict`,
`unprocessable_entity`, `rate_limited`, `provider_reauth_required`,
`provider_upstream_error`, `internal_error`. Always capture `request_id` — it's the key for
support/log correlation.

## Rate limits

IP-based, **60 requests/min** by default (OAuth endpoints are stricter, 10/min). Every
response carries `X-RateLimit-{Limit,Remaining,Reset}`; a `429` includes `Retry-After` —
honor it with backoff rather than hammering.

## The read/write boundary (important)

* **Reads + control plane → REST.** Create/alter tables, manage domains/storage/org, list
  and read small result sets.
* **Row DML is NOT REST.** Insert/update/merge/delete go over the gRPC path — **there is no
  REST merge endpoint**. Do writes via the CLI (`eddytor insert`/`merge`), an SDK, or MCP
  write tools. DML guardrail failures return a structured write-error envelope (unique/PK →
  409, domain/check/not-null → 422, type → 400).
* **Bulk reads → Flight SQL**, not REST pagination — see eddytor-flight-sql.

## Gotchas

* A `5xx` without an error envelope is an opaque internal fault — treat it as a generic
  error and retry with backoff; never assume it's a client (4xx) problem.
* Access tokens expire (~15 min). For long-running integrations use an API key, or
  implement refresh — don't pin a single access token.
* `provider_reauth_required` means a delegated storage credential lapsed — the user must
  re-link (see eddytor-provider-oauth), not a retry.
* Discover endpoints/params from `/spec` (OpenAPI); don't guess routes.

## Guidelines

* Generate a client from `/spec` and key error handling off `code` + `request_id`.
* API key for machines, OAuth for users; keep credentials in env/secrets, never in code.
* REST for control + small reads, Flight SQL for bulk reads, gRPC/CLI/SDK/MCP for writes.
