---
name: eddytor-deploy-helm
description: >
  Installs self-hosted Eddytor on any Kubernetes cluster using the published OCI
  Helm chart. Activates for deploying Eddytor on Kubernetes, Helm install, the
  eddytor chart, ingress/TLS, external vs bundled Postgres, or scaling the engine —
  cluster-agnostic. Cloud-specific skills (AKS/GKE/EKS/Hetzner) build on this one.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Deploy Eddytor with Helm (any Kubernetes)

The chart is published as an OCI artifact — **install it directly, never clone or
build**. It defaults to the `:k8s` kubernetes-edition images (the server discovers
engine replicas via DNS). Needs Helm ≥ 3.8 (OCI support).

Chart: `oci://ghcr.io/nordalf/charts/eddytor`

## Default procedure

1. `kubectl create namespace eddytor`
2. Create the secret with the three required values (below).
3. `helm upgrade --install` with your Postgres + object-store + ingress choices.
4. Verify Postgres connectivity and that the server ran migrations (see Verify).
5. Provision the first admin **inside** the server pod.

## Required secrets

Three values are mandatory (validated at boot by preflight):

```bash
kubectl -n eddytor create secret generic eddytor-secrets \
  --from-literal=EDDYTOR_DATABASE_URL="postgres://eddytor:eddytor@eddytor-postgres:5432/eddytor" \
  --from-literal=EDDYTOR_ENCRYPTION_KEY="$(openssl rand -base64 32)" \
  --from-literal=EDDYTOR_API_KEY_SECRET="$(openssl rand -base64 32)"
```

* `EDDYTOR_ENCRYPTION_KEY` must be base64 of exactly 32 bytes; `EDDYTOR_API_KEY_SECRET`
  is 32+ random bytes. Reference it with `--set secrets.existingSecret=eddytor-secrets`.
* For **bundled** Postgres the DB URL host MUST be `eddytor-postgres` and creds MUST match
  `postgres.user/password/database` (defaults `eddytor`/`eddytor`/`eddytor`). A mismatch is
  the #1 install failure.

## Install

```bash
helm upgrade --install eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set secrets.existingSecret=eddytor-secrets \
  --set postgres.bundled=true \           # eval Postgres; auto-forces waitForDb on
  --set garage.bundled=true \             # eval in-cluster S3 so you can create tables
  --set config.publicUrl="https://eddytor.example.com" \
  --set ingress.enabled=true --set ingress.host=eddytor.example.com
```

### Production vs evaluation

* **Evaluation:** `postgres.bundled=true` + `garage.bundled=true`. Single replica, no HA,
  no backups — never for data you keep.
* **Production:** leave both false, put the external DB URL in the secret, and register a
  real object store after install (Azure Blob / S3 / GCS — see eddytor-storage-registration).
  **Leave `waitForDb` off for an external DB** — its init container probes the bundled
  Service name `eddytor-postgres`, not your DB URL, so enabling it with an external DB hangs
  the pods in `Init`. The server runs its own preflight + advisory-locked migration anyway.
  `waitForDb` is only meaningful with bundled Postgres (where it auto-forces on).

### Reaching the server

* **Ingress + DNS:** `--set ingress.enabled=true --set ingress.host=... --set ingress.className=...`
  and `--set config.publicUrl=https://that-host`. Needs an ingress controller in the cluster.
* **Raw public IP, no DNS:** `--set server.service.type=LoadBalancer`, then read the
  assigned IP and re-run `helm upgrade --reuse-values --set config.publicUrl=http://<IP>:8080`
  (publicUrl drives cookies/OAuth/JWKS, so it must match the address users hit — but the LB
  IP isn't known until provisioned, hence the two-step).
* **No ingress yet:** `kubectl -n eddytor port-forward svc/eddytor-server 8080:8080` and
  install with `--set config.publicUrl=http://localhost:8080`.

### Optional web UI

`--set ui.enabled=true`. The UI runs on its **own origin** (port 3000); set
`--set ui.origin=<public-url>` so the chart registers `{ui.origin}/auth/callback` as a
server redirect URI. Expose it like the server (its own LoadBalancer/ingress).

## Verify (REQUIRED — proves the DB handshake)

```bash
kubectl -n eddytor rollout status statefulset/eddytor-postgres --timeout=180s   # bundled only
kubectl -n eddytor rollout status deploy/eddytor-server --timeout=300s
kubectl -n eddytor logs deploy/eddytor-server | grep -iE "preflight|migrat|listening"
helm test eddytor -n eddytor          # hits /healthz on server + engine
```

## First admin

```bash
kubectl -n eddytor exec deploy/eddytor-server -- \
  eddytoradm setup --email you@example.com --org "Your Org"
# headless key instead of browser login:
kubectl -n eddytor exec deploy/eddytor-server -- \
  eddytoradm create-api-key --email you@example.com
```

## Gotchas

* **Install from the OCI chart only** — cloning/building is a maintainer-only path.
* Use the `:k8s` images (chart default). `:latest` is the community/compose edition and
  will not do DNS engine discovery.
* The engine Services are fixed names (`eddytor-engine` / `eddytor-engine-headless`) — run
  **one release per namespace**; don't rename them.
* **Bundled Postgres on a real CSI volume (azuredisk/EBS/PD) may crash-loop** with
  `chown ... Operation not permitted` if the chart drops all Linux capabilities. If you
  see that, patch the StatefulSet to add `CHOWN,FOWNER,DAC_OVERRIDE,SETGID,SETUID`
  (or use external Postgres). Local hostPath clusters don't hit it.
* Server is the authoritative migrator (advisory-locked at boot); the engine sets
  `EDDYTOR_SKIP_MIGRATIONS=true`. If the server stays in `Init`, `waitForDb` can't reach
  Postgres — re-check the DB URL host/creds.

## Guidelines

* Scale the (stateless) engine any time: `kubectl -n eddytor scale deploy/eddytor-engine --replicas=N`.
* Trim replicas on small clusters: `--set engine.replicas=1 --set server.replicas=1 --set engine.autoscaling.enabled=false`.
* Cloud specifics (cluster creation, managed Postgres, identity-based storage, ingress)
  live in the per-cloud skills: eddytor-deploy-aks / -gke / -eks / -hetzner.
