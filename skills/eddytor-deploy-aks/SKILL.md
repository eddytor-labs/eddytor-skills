---
name: eddytor-deploy-aks
description: >-
  Deploys self-hosted Eddytor on Azure Kubernetes Service (AKS). Builds on
  eddytor-deploy-helm — only the Azure-specific deltas live here. Activates for
  AKS, Azure Kubernetes, az aks, deploy on Azure, Azure Database for PostgreSQL
  Flexible Server, Azure Blob storage, or quota/vCPU errors during cluster
  creation.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Deploy Eddytor on AKS

This is the **Azure delta** over [eddytor-deploy-helm](../eddytor-deploy-helm/SKILL.md).
The chart install, required secrets, verify, and first-admin steps are identical —
**follow eddytor-deploy-helm for those** and only apply the AKS-specific pieces below.
Install from the OCI chart only (`oci://ghcr.io/nordalf/charts/eddytor`); never clone or build.

Needs `az`, `kubectl`, `helm` ≥ 3.8, `openssl`, and a subscription that can create
resource groups + AKS clusters.

## Default procedure

1. Create the cluster (resource group → `az aks create` → `get-credentials`).
2. Provision Postgres — **Azure Database for PostgreSQL Flexible Server** for production
   (or `postgres.bundled=true` for evaluation, with the azuredisk caveat below).
3. Create the secret + `helm upgrade --install` per eddytor-deploy-helm, with the
   Azure conn string and `--set waitForDb.enabled=true` when external.
4. Verify + first admin per eddytor-deploy-helm.
5. Register Azure Blob as the object store (see eddytor-storage-registration).

## Tool usage examples (CLI commands)

### Create the cluster

```bash
az group create --name eddytor-rg --location westeurope

az aks create \
  --resource-group eddytor-rg --name eddytor-aks --location westeurope \
  --node-count 3 --node-vm-size Standard_D4as_v4 \
  --enable-managed-identity --generate-ssh-keys

az aks get-credentials --resource-group eddytor-rg --name eddytor-aks --overwrite-existing
kubectl get nodes        # all Ready
```

Engine requests ~1 CPU / 2Gi and server ~200m / 512Mi (×2 each by default), so three
4-vCPU nodes give headroom. Smaller demo: 2 nodes + trim replicas (see Guidelines).

### Managed Postgres (production)

```bash
az postgres flexible-server create \
  --resource-group eddytor-rg --name eddytor-pg --location westeurope \
  --admin-user eddytor --admin-password '<STRONG_PASSWORD>' \
  --tier Burstable --sku-name Standard_B2s \
  --version 16 --storage-size 32 --database-name eddytor \
  --public-access 0.0.0.0    # demo: allow Azure services. Prefer VNet/Private Link for prod.
```

Azure requires TLS, so the conn string **must** end with `?sslmode=require`, and the host
is the FQDN Azure prints (`eddytor-pg.postgres.database.azure.com`). Put it in the secret
that eddytor-deploy-helm creates, and pass `--set waitForDb.enabled=true`:

```bash
kubectl -n eddytor create secret generic eddytor-secrets \
  --from-literal=EDDYTOR_DATABASE_URL="postgres://eddytor:<STRONG_PASSWORD>@eddytor-pg.postgres.database.azure.com:5432/eddytor?sslmode=require" \
  --from-literal=EDDYTOR_ENCRYPTION_KEY="$(openssl rand -base64 32)" \
  --from-literal=EDDYTOR_API_KEY_SECRET="$(openssl rand -base64 32)"
```

### Object store — Azure Blob (after install)

The user needs object storage to create tables. Register Azure Blob as a storage
connection after install — prefer **managed identity** (`use_msi`), or a service principal.
See **eddytor-storage-registration** for the `register_az_storage` parameters (do not
restate them here). For evaluation only, `--set garage.bundled=true` gives an in-cluster S3.

### Raw public IP, no DNS (two-step)

The LB IP isn't known until Azure provisions it, so install first, then reconcile the URL:

```bash
# 1. install with the server on a LoadBalancer (per eddytor-deploy-helm Install)
helm upgrade --install eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set secrets.existingSecret=eddytor-secrets \
  --set waitForDb.enabled=true \
  --set server.service.type=LoadBalancer \
  --set config.publicUrl="http://pending:8080"

# 2. read the assigned IP, then set publicUrl to it
kubectl -n eddytor get svc eddytor-server -w     # wait for EXTERNAL-IP
helm upgrade eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor --reuse-values \
  --set config.publicUrl="http://<SERVER_EXTERNAL_IP>:8080"
```

The same two-step applies to the UI on its own LoadBalancer when `ui.enabled=true`:
`--set ui.service.type=LoadBalancer`, then `--reuse-values --set ui.origin="http://<UI_EXTERNAL_IP>:3000"`.

### Teardown

```bash
az group delete --name eddytor-rg --yes --no-wait
```

## Gotchas

* **vCPU quota = 0 for a VM family.** A region/subscription may have **zero** quota for a
  family (e.g. `Standard_DSv5`) and `az aks create` fails with `InsufficientVCPUQuota`.
  Check headroom and pick a family that has it before retrying:

  ```bash
  az vm list-usage --location westeurope -o table   # Usage vs Limit per family
  ```

  `Standard_D4as_v4` / the DASv4 family is a safe fallback in most regions; otherwise
  request a quota increase in the portal (Subscriptions → Usage + quotas).
* **Bundled Postgres crash-loops on azuredisk** with `chown: Operation not permitted` — the
  chart drops all Linux capabilities and the real CSI volume needs them at init. Two fixes:
  use the managed Flexible Server (recommended), **or** patch the StatefulSet to add the
  caps back:

  ```bash
  kubectl -n eddytor patch statefulset eddytor-postgres --type=json -p='[{"op":"add",
    "path":"/spec/template/spec/containers/0/securityContext/capabilities/add",
    "value":["CHOWN","FOWNER","DAC_OVERRIDE","SETGID","SETUID"]}]'
  ```

  Local hostPath clusters don't hit this — it's specific to real CSI volumes.
* **Azure Postgres needs `?sslmode=require`** in the conn string, and the AKS egress IP
  (or VNet) must be allowed through the server firewall, or `waitForDb` blocks server boot.
* `config.publicUrl` (and `ui.origin`) MUST equal the address the browser actually hits, or
  OAuth redirects and cookies break — hence the two-step on the raw-IP path.

## Guidelines

* Cross-reference, don't duplicate: chart flags, secret format, verify, and first-admin all
  live in **eddytor-deploy-helm**; storage params in **eddytor-storage-registration**.
* Engine Services are fixed-named (`eddytor-engine` / `eddytor-engine-headless`) — **one
  release per namespace**.
* Small node pool? Trim replicas: `--set engine.replicas=1 --set server.replicas=1 --set engine.autoscaling.enabled=false`.
* Ingress + TLS (shared L7 public IP via the AKS app-routing add-on or ingress-nginx +
  cert-manager) is the production-grade exposure; raw LoadBalancer IPs are plaintext L4.
