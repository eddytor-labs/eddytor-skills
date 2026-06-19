---
name: eddytor-deploy-object-store
description: >
  Chooses and wires the object store that backs a self-hosted Eddytor deployment —
  bundled Garage for evaluation vs Azure Blob / S3 / GCS for production — and registers
  it right after install. Activates for "where does Eddytor store data", backing store,
  bundled Garage, S3/Blob/GCS for production, or object-store setup during deploy.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Wiring Eddytor's Object Store (deploy time)

Eddytor stores every Delta table in an object store. A deployment needs at least one,
and the choice is install-time: a bundled evaluation store, or a real cloud store you
register after the pods are healthy.

## The decision

* **Bundled Garage (evaluation only):** `--set garage.bundled=true` deploys a
  single-node in-cluster S3 so you can create tables immediately. No HA, demo
  credentials — never for data you keep.
* **External store (production):** leave `garage.bundled` false and register Azure Blob,
  S3, or GCS as a storage configuration **after** install. This is the durable path.

The chart only stands up (or omits) the bundled store; pointing Eddytor at a real bucket
is a runtime registration, not a Helm flag.

## Default procedure (production)

1. Install per eddytor-deploy-helm with `garage.bundled` left false.
2. Once the server is healthy, register the real store (full options in
   eddytor-storage-registration):

```
register_az_storage(account_name="acme", container="mdm", use_msi=true)   # Azure
register_s3_storage(bucket_name="mdm-prod", region="eu-west-1", ...)       # AWS
register_gcs_storage(bucket_name="mdm-prod", service_account_key="{...}")  # GCP
```

3. `list_storage_configs` to capture the `config_id`, then create/import tables.

## Prefer workload identity over static keys

Each cloud lets the pod authenticate without long-lived secrets — use it:

* **Azure:** `use_msi=true` with a managed identity / workload identity on the node pool
  (or a service principal via `client_id`/`client_secret`/`tenant_id`).
* **AWS:** IRSA — attach an IAM role to the Eddytor service account; register S3 without
  `access_key_id`/`secret_key` where the environment provides credentials.
* **GCP:** GKE Workload Identity bound to a service account with bucket access.

See the per-cloud deploy skills (eddytor-deploy-aks / -eks / -gke) for binding the
identity, and eddytor-storage-registration for the tool parameters.

## Gotchas

* Bundled Garage `accessKey/secretKey/rpcSecret/adminToken` are **demo placeholders** —
  override them (`--set garage.accessKey=...`) for anything beyond a throwaway, or just
  use an external store.
* Switching from bundled to external later means re-registering and recreating/moving
  tables — pick external up front for production.
* A registered store with `discover_files=true` exposes standalone Parquet/CSV as
  **read-only** tables; only Delta tables are writable.
* The bundled store is in-cluster — its data lives on a single PVC with no backups.

## Guidelines

* Evaluation → bundled Garage. Anything you keep → external bucket + workload identity.
* Register the store immediately after the first admin so the instance is usable.
