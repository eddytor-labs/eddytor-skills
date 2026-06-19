---
name: eddytor-deploy-gke
description: >-
  Deploys self-hosted Eddytor on Google Kubernetes Engine. Activates for GKE,
  Google Kubernetes Engine, gcloud, deploy on GCP, GKE Autopilot, Cloud SQL for
  PostgreSQL, GCS bucket storage, Workload Identity, or a Google-managed
  ingress certificate — even without the phrase "deploy on GKE."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Deploy Eddytor on GKE

GKE-specific deltas on top of **eddytor-deploy-helm** (the cluster-agnostic base).
That base owns the chart install, the three required secrets, the verify steps,
and first-admin provisioning — follow it for all of those and only apply the GKE
specifics below. Install from the OCI chart `oci://ghcr.io/nordalf/charts/eddytor`;
never clone or build.

## Default procedure

1. Create/get a cluster, then fetch credentials (below).
2. Stand up **Cloud SQL for PostgreSQL** and build the `EDDYTOR_DATABASE_URL`
   (with `?sslmode=require`) into the secret from the base skill.
3. `helm upgrade --install` with `--set waitForDb.enabled=true` for external DB,
   plus your ingress/LoadBalancer choice.
4. Verify + provision first admin **per the base skill**.
5. Register a **GCS** bucket after install — see eddytor-storage-registration.
   Prefer Workload Identity over a static key.

## Tool usage examples

### Cluster (Autopilot recommended)

```bash
# Autopilot — Google manages nodes; Workload Identity is on by default
gcloud container clusters create-auto eddytor --region=europe-west1

# OR a Standard cluster with an explicit node pool + Workload Identity
gcloud container clusters create eddytor --region=europe-west1 \
  --num-nodes=2 --machine-type=e2-standard-4 \
  --workload-pool=PROJECT_ID.svc.id.goog

gcloud container clusters get-credentials eddytor --region=europe-west1
kubectl create namespace eddytor
```

### Cloud SQL for PostgreSQL

Connect via **private IP** (simplest with a VPC-native cluster) or the **Cloud SQL
Auth Proxy** sidecar. Put the resulting connection string in the secret with TLS
required:

```bash
# inside the base skill's `kubectl create secret generic eddytor-secrets`:
--from-literal=EDDYTOR_DATABASE_URL="postgres://eddytor:PW@10.x.x.x:5432/eddytor?sslmode=require"
```

Install with the external-DB wait gate enabled:

```bash
helm upgrade --install eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set secrets.existingSecret=eddytor-secrets \
  --set waitForDb.enabled=true \
  --set config.publicUrl="https://eddytor.example.com" \
  --set ingress.enabled=true --set ingress.host=eddytor.example.com
```

### GCS storage via Workload Identity (preferred)

Bind the Eddytor Kubernetes ServiceAccount to a GCP service account that holds
`roles/storage.objectAdmin` on the bucket — no static key in the cluster:

```bash
gcloud iam service-accounts create eddytor-gcs
gsutil iam ch \
  serviceAccount:eddytor-gcs@PROJECT_ID.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://your-eddytor-bucket

# allow the K8s SA used by the engine to impersonate it
gcloud iam service-accounts add-iam-policy-binding \
  eddytor-gcs@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[eddytor/eddytor-engine]"

kubectl -n eddytor annotate serviceaccount eddytor-engine \
  iam.gke.io/gcp-service-account=eddytor-gcs@PROJECT_ID.iam.gserviceaccount.com
```

Then register the bucket with `register_gcs_storage` (params in
eddytor-storage-registration). With Workload Identity the pod gets credentials
ambiently — no key file needed.

### Ingress / TLS

```bash
# GKE Ingress (GCE ingress class) + Google-managed cert
--set ingress.enabled=true --set ingress.host=eddytor.example.com \
--set ingress.className=gce --set ingress.tls=true
```

Raw external IP, no DNS — use a LoadBalancer, then do the **two-step publicUrl
update from the base skill** (the LB IP is unknown until provisioned):

```bash
--set server.service.type=LoadBalancer
# read EXTERNAL-IP, then:
helm upgrade --reuse-values eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set config.publicUrl=http://<EXTERNAL-IP>:8080
```

### Teardown

```bash
gcloud container clusters delete eddytor --region=europe-west1
```

## Gotchas

* **Bundled Postgres crash-loops on the GCE persistent-disk CSI** with
  `chown: Operation not permitted` — the chart drops all Linux caps and the PD
  volume needs them. Either patch the StatefulSet to add caps
  `CHOWN,FOWNER,DAC_OVERRIDE,SETGID,SETUID`, or (recommended) use **Cloud SQL**
  and skip `postgres.bundled`. Local hostPath clusters don't hit this.
* Cloud SQL public IPs reject plaintext — keep `?sslmode=require` in the DB URL,
  or the server's preflight DB handshake fails and it stays in `Init`.
* Google-managed certs only issue once the Ingress has a stable IP **and** DNS
  resolves to it; the cert sits `Provisioning` until both are true.
* The engine Services are fixed names (`eddytor-engine` / `eddytor-engine-headless`)
  → **one release per namespace**; the Workload Identity binding above targets
  the `eddytor-engine` KSA specifically.
* Autopilot rejects pods that don't fit its resource model — if a pod is stuck
  `Pending`, check it requests CPU/memory within Autopilot limits.

## Guidelines

* **Cloud SQL for production, bundled only for throwaway eval.** For eval that
  must stay on bundled Postgres, apply the caps patch above.
* Allowed chart flags only: `secrets.existingSecret`, `postgres.bundled`,
  `postgres.password`, `waitForDb.enabled`, `garage.bundled`, `config.publicUrl`,
  `config.cookieDomain`, `ingress.enabled`, `ingress.host`, `ingress.className`,
  `ingress.tls`, `server.service.type`, `server.replicas`, `engine.replicas`,
  `engine.autoscaling.enabled`, `ui.enabled`, `ui.origin`, `ui.service.type`.
* Prefer Workload Identity to mounted GCS keys — fewer secrets to rotate.
* Optional web UI runs on its own origin: `--set ui.enabled=true --set ui.origin=<url>`
  (expose with its own `ui.service.type` LoadBalancer/ingress).
