---
name: eddytor-provider-oauth
description: >-
  Configures per-organisation Azure / Google OAuth applications so Eddytor can
  reach cloud storage with delegated (OAuth) credentials instead of static keys.
  Activates for "provider OAuth app", delegated storage access, Azure app
  registration, Google OAuth client, or "connect Azure/Google with OAuth instead
  of keys" — even without the word "provider."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Provider OAuth Apps (delegated Azure / Google storage)

A **provider OAuth app** holds an organisation's own Azure AD or Google Cloud
client credentials so Eddytor can obtain **delegated (OAuth) access** to a user's
cloud storage — the alternative to handing Eddytor static access keys. The client
id / secret (and, for Azure, the tenant) are configured **per organisation in the
database** (table `eddytor.provider_oauth_apps`). There is **no environment
variable** for these — the per-org DB row replaced the old global env-var
credentials.

Managing apps requires the **`provider_apps:write`** scope; reading uses
**`provider_apps:read`**. Both are granted **only to the Admin role**
(`ADMIN_ADDITIONAL_SCOPES`); Builder, Editor, and Viewer have neither. An API key
caller additionally needs the scope in its intersection.

This is distinct from two other things:
- **SSO login federation** (`sso_connections`) — how users *sign in*, not storage.
- **Static-key storage registration** (skill `eddytor-storage-registration`) —
  the `register_az_storage` / `register_gcs_storage` step that actually attaches a
  container/bucket. Configure the provider OAuth app **first**, then run that
  registration so it uses the org's delegated identity instead of keys.

## Default procedure

1. In the org's Azure AD (App registration) or Google Cloud console, create an
   OAuth client and copy its **client id** and **client secret** (Azure also has a
   **directory/tenant id**).
2. As an **Admin** (scope `provider_apps:write`), register it on the org via the
   REST API: `POST /v1/organisations/{org_id}/provider-apps/{provider}` with
   `client_id`, `client_secret`, and optional `tenant`. `{provider}` is `azure`
   or `google` (GitHub is rejected).
3. Verify with `GET /v1/organisations/{org_id}/provider-apps` (scope
   `provider_apps:read`). The secret is **never** returned — only client id,
   tenant, and timestamps.
4. Register the storage (skill `eddytor-storage-registration`) — `AzureTokenProvider`
   / `GoogleTokenProvider` will resolve this org's app at runtime and mint
   delegated tokens.

## Tool usage examples

Configuration is via the org REST API (no dedicated MCP tool, no CLI subcommand).
Authenticate as an Admin and target your `org_id`.

Register/replace an Azure app:

```
POST /v1/organisations/{org_id}/provider-apps/azure
{
  "client_id": "11111111-2222-3333-4444-555555555555",
  "client_secret": "<azure-app-secret>",
  "tenant": "contoso.onmicrosoft.com"      // omit for the multi-tenant /organizations authority
}
```

Register/replace a Google app (`tenant` ignored):

```
POST /v1/organisations/{org_id}/provider-apps/google
{ "client_id": "...apps.googleusercontent.com", "client_secret": "<google-client-secret>" }
```

List or fetch (secret omitted), then remove:

```
GET    /v1/organisations/{org_id}/provider-apps            # all configured providers
GET    /v1/organisations/{org_id}/provider-apps/azure      # one provider, or null if unset
DELETE /v1/organisations/{org_id}/provider-apps/azure      # provider_apps:write
```

## Gotchas

- **No env var.** Don't look for `EDDYTOR_AZURE_CLIENT_*` / `EDDYTOR_GOOGLE_CLIENT_*`
  — they're gone. Credentials live only in the per-org `provider_oauth_apps` row.
- **Admin only.** `provider_apps:read` / `provider_apps:write` are granted solely to
  Admin. A non-Admin (or an API key lacking the scope) gets `forbidden`.
- **Unconfigured org is a 422, not a crash.** When no app exists,
  `AzureTokenProvider::new` / `GoogleTokenProvider::new` return
  `EddytorError::ProviderNotConfigured` → HTTP **422** (`unprocessable_entity`).
  Never a panic. If delegated storage fails with 422, the org simply hasn't
  registered the provider app yet.
- **Upsert, one per provider.** A second `POST` for the same provider replaces the
  row (`ON CONFLICT (organisation_id, provider)`). Changing the `client_id`
  invalidates previously-issued user refresh tokens, so users must re-auth.
- **GitHub is not a storage provider.** `POST .../provider-apps/github` is rejected.
- **Secret is write-only.** GET/list never echo the client secret; to change it,
  POST again.

## Guidelines

- Configure the provider app **before** registering OAuth-backed Azure/Google
  storage; the registration step depends on the delegated identity existing.
- Treat the client secret like any credential — it's vaulted in Eddytor's secret
  store on write and can't be read back.
- Use a **dedicated** Azure AD / Google Cloud app per org so revoking one org's
  credentials never affects another.
- For Azure single-tenant apps set `tenant`; omit it to use the multi-tenant
  `/organizations` authority.
