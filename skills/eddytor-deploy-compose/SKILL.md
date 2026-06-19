---
name: eddytor-deploy-compose
description: >
  Installs self-hosted Eddytor on a single host with Docker Compose using the public
  prebuilt images. Activates for self-hosting on one VM, docker compose, the
  get.eddytor.dev installer, single-server Eddytor, or "no Kubernetes" — the simplest
  deployment path.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Deploy Eddytor with Docker Compose (single host)

The simplest self-host path: one VM running server + engine + Postgres via Compose,
using the **public community images** (`ghcr.io/nordalf/...`, `:latest` / `:VERSION`).
Consume prebuilt artifacts — **never clone or build** (that's a maintainer-only path).
For horizontal scale or managed cloud, use the Kubernetes skills instead (eddytor-deploy-helm).

## Default procedure (installer)

```bash
curl get.eddytor.dev | sh
```

The installer downloads `docker-compose.yml` + `install.sh` from the GitHub release,
prompts for configuration, writes `.env`, and brings the stack up. Then create the first
admin (see below).

## Manual procedure

1. Fetch `deploy/selfhost/docker-compose.yml` from the GitHub release (don't build it).
2. Create `.env` with the three required secrets + your public URL:

```
EDDYTOR_DATABASE_URL=postgres://eddytor:eddytor@postgres:5432/eddytor
EDDYTOR_ENCRYPTION_KEY=<base64 of 32 random bytes>   # openssl rand -base64 32
EDDYTOR_API_KEY_SECRET=<32+ random bytes>            # openssl rand -base64 32
EDDYTOR__SERVER__PUBLIC_URL=https://eddytor.example.com
```

3. `docker compose up -d`
4. Provision the first admin inside the server container:

```bash
docker compose exec eddytor-server eddytoradm setup --email you@example.com --org "Your Org"
```

## Optional web UI

The UI is an opt-in Compose profile. The installer knob `EDDYTOR_WITH_UI=true` writes
`COMPOSE_PROFILES=ui`, `EDDYTOR_UI_VERSION`, and `EDDYTOR__SERVER__WEB_REDIRECT_URIS` into
`.env`. The UI image (`ghcr.io/nordalf/eddytor-ce-ui`) versions on its own cadence.

## TLS

Binaries bind plaintext (h2c); terminate TLS at the edge — a Caddy overlay (see the
shipped `community.tls.toml`) or any reverse proxy in front of port 8080.

## Gotchas

* Use the **community** images (`:latest` / `:VERSION`). The `:k8s` tags are for the Helm
  chart (DNS engine discovery) and won't fit the Compose `engine.host` direct-TCP wiring.
* `get.eddytor.dev` serves the published release asset, not your local checkout — fixes
  land only when a new version is tagged.
* The server→engine connection uses `engine.host` (the `eddytor-engine` service in
  Compose) over plaintext on the host network — keep the ports off the public internet;
  expose only the edge TLS proxy.
* The same three secrets are mandatory; the server preflight fails fast if any is missing.

## Guidelines

* Single host = single point of failure. For HA / scale-out, move to Kubernetes
  (eddytor-deploy-helm and the per-cloud skills).
* Back up the Postgres volume and keep `EDDYTOR_ENCRYPTION_KEY` safe — losing it makes
  stored secrets unrecoverable.
