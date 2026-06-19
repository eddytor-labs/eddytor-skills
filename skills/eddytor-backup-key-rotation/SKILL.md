---
name: eddytor-backup-key-rotation
description: >-
  Backup, restore, and secret rotation for self-hosted Eddytor. Activates for
  backup, restore, disaster recovery, key rotation, rotate secrets, the
  encryption key (EDDYTOR_ENCRYPTION_KEY), the API key secret
  (EDDYTOR_API_KEY_SECRET), OAuth signing keys, Postgres backups, or recovering
  a lost admin key.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Backup & key rotation (self-hosted Eddytor)

Eddytor's recoverable state is three things: the **Postgres database** (all rows,
including encrypted secret handles and OAuth signing keys), the **`EDDYTOR_ENCRYPTION_KEY`**
master key, and the **`EDDYTOR_API_KEY_SECRET`** HMAC secret. Back up all three together —
the DB is useless without the keys that decrypt/verify against it.

The three required secrets are set wherever you deploy: in `.env` for compose, or the
`eddytor-secrets` k8s Secret for Helm (see eddytor-deploy-helm / the per-cloud deploy
skills for exactly where).

## Default procedure

**Back up**

1. Back up Postgres (see Postgres backups below — bundled has NONE).
2. Back up the secrets: `.env` (compose) or the `eddytor-secrets` k8s Secret (Helm). Store
   `EDDYTOR_ENCRYPTION_KEY` and `EDDYTOR_API_KEY_SECRET` somewhere durable — losing the
   encryption key makes every stored secret unrecoverable, and a wrong key fails server
   boot with `secret decrypt failed`.
3. Snapshot Postgres before any upgrade — migrations are forward-only.

**Restore**

1. Restore the Postgres data.
2. Restore the *same* `EDDYTOR_ENCRYPTION_KEY` (decrypts existing handles) and
   `EDDYTOR_API_KEY_SECRET` (keeps issued API keys valid).
3. Start the stack; the server runs migrations and boots. A key mismatch surfaces as
   boot failure or `provider_reauth_required` on stored credentials.

**Rotate the encryption key** (treat as a blast-radius reset, not routine hygiene — the
store is stateless and the handle IS the ciphertext, so there is no per-secret revocation):

1. Generate a new key: `openssl rand -base64 32`.
2. Schedule a maintenance window.
3. For each user/org, re-enter every stored secret — re-link OAuth providers, re-create
   storage credentials, re-add AI credentials. There is **no scriptable migration**: by
   design the old key cannot decrypt under the new master.
4. Swap `EDDYTOR_ENCRYPTION_KEY` and restart all replicas.
5. Old handles in the DB now fail to decrypt and surface as `provider_reauth_required` /
   similar errors; users re-enter on next use.

**Rotate the API-key secret** — this **invalidates ALL existing API keys** (they are
HMAC-hashed with this secret). After swapping `EDDYTOR_API_KEY_SECRET` and restarting,
re-mint every key with `eddytoradm create-api-key` inside the server pod/container.

**OAuth signing keys (ES256)** rotate on their own via `SigningKeyStore::rotate()` with a
one-TTL grace window (the old key still verifies issued tokens until the window closes).
They live in the DB (`eddytor.oauth_signing_keys`), not in env — so a DB backup carries
them; nothing to rotate by hand.

## Tool usage examples

```bash
# Generate a fresh 32-byte key (encryption key or API-key secret)
openssl rand -base64 32

# Back up an external/managed Postgres
pg_dump "$EDDYTOR_DATABASE_URL" > eddytor-$(date +%F).sql

# Re-mint an API key after rotating EDDYTOR_API_KEY_SECRET (compose)
docker compose exec eddytor-server eddytoradm create-api-key --email you@example.com
# Helm
kubectl -n eddytor exec deploy/eddytor-server -- \
  eddytoradm create-api-key --email you@example.com

# Lost-key recovery: mint a fresh admin key (no one-shot token to recover)
docker compose exec eddytor-server eddytoradm create-api-key --email you@example.com
# Or just sign in again — the magic link is re-requestable:
eddytor login
# With SMTP unset, the link is in the logs:
docker compose logs --tail 50 eddytor-server | grep -i 'sign in to eddytor'
```

## Gotchas

- **`EDDYTOR_ENCRYPTION_KEY` (AES-256-GCM-SIV master key) is the only thing that can decrypt
  stored secrets.** The handle is the ciphertext; `delete()` is a no-op and `update()` does
  not invalidate prior handles. Any leaked handle decrypts until you rotate the master key —
  rotation is the only revocation, and it invalidates every ciphertext at once.
- **Rotating `EDDYTOR_API_KEY_SECRET` invalidates ALL API keys** — every integration breaks
  until keys are re-minted. Don't do it casually.
- **Bundled Postgres has NO backups.** The Helm/compose bundled DB is a single PVC for
  evaluation only — single replica, no HA, no backups. Never put data you keep on it.
- **For production, use the managed database's backups** (e.g. cloud PITR) or run your own
  `pg_dump` schedule. The DB is the source of truth for rows, secret handles, and OAuth
  signing keys.
- **Migrations are forward-only** — snapshot Postgres before every upgrade.
- **Nothing to recover for a lost API key** — there is no one-shot token. The auth boundary
  is exec access; re-mint with `eddytoradm create-api-key` as often as needed. `eddytoradm
  setup` is idempotent (no-op once an admin exists), so it can't be used to seize a second admin.

## Guidelines

- Back up the DB **and** both env secrets together; one without the other is unrecoverable.
- Keep `EDDYTOR_ENCRYPTION_KEY` and `EDDYTOR_API_KEY_SECRET` stable across restarts and
  restores; only change them as deliberate rotations with the consequences above.
- Where the secrets live depends on the deploy: `.env` (compose) or `eddytor-secrets`
  (Helm) — see eddytor-deploy-helm and the per-cloud deploy skills.
- OAuth signing-key rotation is automatic with a grace window; leave it to the server.
