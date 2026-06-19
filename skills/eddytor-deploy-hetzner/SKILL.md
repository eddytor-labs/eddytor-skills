---
name: eddytor-deploy-hetzner
description: >-
  Deploys self-hosted Eddytor on Hetzner Cloud — the budget path. Hetzner has no
  managed Kubernetes, so you bring your own k3s (hetzner-k3s or k3sup) plus the
  Hetzner Cloud Controller Manager and hcloud-csi driver, then install the chart.
  Activates for Hetzner, hcloud, hetzner-k3s, k3s on Hetzner, Hetzner Object
  Storage, Hetzner Load Balancer, or "cheapest way to self-host Eddytor."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Deploy Eddytor on Hetzner Cloud (budget self-host)

The cheapest credible self-host: a k3s cluster on Hetzner Cloud servers, the
chart on top. **Hetzner has no managed Kubernetes and no managed Postgres**, so
you provision both. The chart install / secret / verify / first-admin steps are
identical to **eddytor-deploy-helm** — read that first. This skill only covers
the Hetzner-specific deltas: cluster, controllers, Postgres, object store, LB.

## Default procedure

1. Provision a k3s cluster on Hetzner Cloud (hetzner-k3s, below).
2. Install `hcloud-cloud-controller-manager` (LoadBalancer support) and
   `hcloud-csi` (PersistentVolumes on Hetzner Cloud Volumes).
3. Create the secret and install the chart per **eddytor-deploy-helm** — with the
   Hetzner Postgres + LB choices below.
4. Verify the DB handshake and provision the first admin (base skill, unchanged).
5. After install, register a Hetzner Object Storage bucket as your store
   (eddytor-storage-registration).

## Tool usage examples

### Cluster: hetzner-k3s (the common path)

Provisions a full k3s cluster on Hetzner Cloud servers from one config file.

```bash
export HCLOUD_TOKEN=<your-hetzner-cloud-api-token>
# cluster_config.yaml lists location, server types, master/worker pools, SSH key
hetzner-k3s create --config cluster_config.yaml
export KUBECONFIG=./kubeconfig
```

Plain-server alternative: create hcloud servers and bootstrap with `k3sup install`.

### Hetzner controllers (required for LB + PVs)

```bash
kubectl -n kube-system create secret generic hcloud --from-literal=token=$HCLOUD_TOKEN
helm repo add hcloud https://charts.hetzner.cloud && helm repo update
helm install hccm   hcloud/hcloud-cloud-controller-manager -n kube-system
helm install hcsi   hcloud/hcloud-csi -n kube-system   # provides the hcloud-volumes StorageClass
```

### Install (Hetzner deltas only — rest is eddytor-deploy-helm)

```bash
helm upgrade --install eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set secrets.existingSecret=eddytor-secrets \
  --set postgres.bundled=true \
  --set postgres.storageClassName=hcloud-volumes \   # bundled DB on a Hetzner Volume
  --set config.publicUrl="http://PLACEHOLDER:8080" \
  --set server.service.type=LoadBalancer             # provisions a Hetzner Load Balancer
```

### Reaching the server (LB has no IP until provisioned — two-step)

```bash
kubectl -n eddytor get svc eddytor-server -w   # wait for EXTERNAL-IP
helm upgrade eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor --reuse-values \
  --set config.publicUrl="http://<LB-IP>:8080"
```

Or install `ingress-nginx` (it gets its own Hetzner LB), point DNS at that LB,
and use `--set ingress.enabled=true --set ingress.host=eddytor.example.com`
+ `--set config.publicUrl=https://eddytor.example.com` instead.

### External Postgres (production on Hetzner)

No managed Postgres exists. Run/operate your own on a Hetzner server or VM, then:

```bash
# DB URL goes in the secret (EDDYTOR_DATABASE_URL); then:
helm upgrade --install eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set secrets.existingSecret=eddytor-secrets \
  --set postgres.bundled=false \
  --set waitForDb.enabled=true \
  --set server.service.type=LoadBalancer
```

### Teardown

```bash
hetzner-k3s delete --config cluster_config.yaml
# or, if you built it by hand: delete the hcloud servers, Volumes, and Load Balancers
# (orphaned Volumes/LBs keep billing — check the Hetzner console).
```

## Gotchas

* **Bundled Postgres crash-loops on hcloud-csi** with `chown: Operation not
  permitted` — the chart drops all Linux caps and the hcloud Volume mount needs
  them. Fix: patch the StatefulSet to add `CHOWN,FOWNER,DAC_OVERRIDE,SETGID,SETUID`,
  or use external Postgres. (Local hostPath doesn't hit this; a real CSI volume does.)
* **No LoadBalancer without hccm.** `server.service.type=LoadBalancer` stays
  `<pending>` forever until `hcloud-cloud-controller-manager` is installed and the
  `hcloud` token secret exists in `kube-system`.
* **No PersistentVolumes without hcsi.** Bundled Postgres/Garage PVCs stay
  `Pending` until `hcloud-csi` provides a StorageClass — pass it via
  `postgres.storageClassName=hcloud-volumes`.
* **Hetzner Object Storage is S3-compatible** — register it with `register_s3_storage`
  and **set its `endpoint`** (regional, e.g. the fsn1/nbg1/hel1 host); see
  eddytor-storage-registration. Any external S3 store (Backblaze B2, AWS S3) works
  the same way.
* **publicUrl two-step is unavoidable** with a raw LB — the IP isn't known until
  Hetzner provisions it, and publicUrl drives cookies/OAuth/JWKS so it must match
  the address users hit.
* **One release per namespace** — engine Services are fixed-named
  (`eddytor-engine` / `eddytor-engine-headless`). Same rule as the base skill.
* **Install from the OCI chart only.** Never clone or build.

## Guidelines

* Budget posture: `postgres.bundled=true` + `garage.bundled=true` + a single
  cheap server is the cheapest eval — single replica, no HA, no backups, never for
  data you keep. Trim with `--set engine.replicas=1 --set server.replicas=1
  --set engine.autoscaling.enabled=false`.
* Step up to durable on the same cluster: external Postgres
  (`postgres.bundled=false` + `waitForDb.enabled=true`) and Hetzner Object Storage
  instead of bundled Garage.
* Mind the bill: Hetzner Load Balancers, Volumes, and Object Storage are billed
  separately from servers — delete orphans on teardown.
* Everything else (secret contents, verify, first-admin, scaling the stateless
  engine) is identical to **eddytor-deploy-helm** — don't restate it.
