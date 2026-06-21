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

## Gather inputs first (prompt the user — never assume)

Before creating anything, **ask the operator** (use AskUserQuestion) and thread their
answers through every command. The names below are examples, not defaults — do not
hardcode them. Confirm:

- **Subscription** (`az account list -o table` → `az account set`) and **region**.
- **Resource group** name (example `eddytor-rg`) — new or existing.
- **Cluster name** (example `eddytor-aks`); optionally the **node resource group** name.
- **Node size + count** (and whether to autoscale the node pool).
- **Networking — which VNet, if any:**
  - *Use an existing VNet/subnet:* pass `--vnet-subnet-id <subnet-resource-id>` (and
    choose the network plugin, e.g. `--network-plugin azure`). Required for hub-spoke,
    private-endpoint, or peered topologies.
  - *Let AKS create one:* omit the subnet flag (kubenet/managed VNet).
  - *Private API server?* add `--enable-private-cluster`.
- **Kubernetes namespace** (example `eddytor`).

Only proceed once these are settled — the operator owns what gets created.

## Default procedure

1. Create the cluster (resource group → `az aks create` → `get-credentials`).
2. Provision Postgres — **Azure Database for PostgreSQL Flexible Server** for production
   (or `postgres.bundled=true` for evaluation, with the azuredisk caveat below).
   On the managed server, **allow-list the `citext` + `pgcrypto` extensions** before
   first boot (Azure blocks them by default; see Managed Postgres) — otherwise
   migration 0 fails.
3. Create the secret + `helm upgrade --install` per eddytor-deploy-helm with the
   Azure conn string. **Do NOT set `waitForDb.enabled=true` for an external DB**
   (chart bug — see Gotchas); the server runs its own preflight + advisory-locked
   migration, so the gate is unnecessary and would block boot.
4. Verify + first admin per eddytor-deploy-helm.
5. Register Azure Blob as the object store (see eddytor-storage-registration).

## Tool usage examples (CLI commands)

### Create the cluster

```bash
az group create --name eddytor-rg --location westeurope

# Substitute the names/sizes/network the operator chose above.
az aks create \
  --resource-group "$RG" --name "$CLUSTER" --location "$REGION" \
  --node-count "$NODES" --node-vm-size "$VM_SIZE" \
  --enable-managed-identity --generate-ssh-keys
  # existing VNet:   --vnet-subnet-id "$SUBNET_ID" --network-plugin azure
  # private cluster: --enable-private-cluster

az aks get-credentials --resource-group "$RG" --name "$CLUSTER" --overwrite-existing
kubectl get nodes        # all Ready
```

Engine requests ~1 CPU / 2Gi and server ~200m / 512Mi (×2 each by default), so three
4-vCPU nodes give headroom. Smaller demo: 2 nodes + trim replicas (see Guidelines).

### Managed Postgres (production)

Read the admin password into a shell variable instead of typing it as a flag — a
password on the command line leaks into shell history and the process list (`ps`):

```bash
read -rs PGADMIN_PW    # prompts without echoing; or source it from a secret manager

# NOTE: do NOT pass --database-name here — on a standard (non-elastic) Flexible
# Server the current az CLI rejects it ("--database-name can only be used when
# --node-count is present"). Create the server, then add the database separately.
az postgres flexible-server create \
  --resource-group "$RG" --name eddytor-pg --location "$REGION" \
  --admin-user eddytor --admin-password "$PGADMIN_PW" \
  --tier Burstable --sku-name Standard_B2s \
  --version 16 --storage-size 32 \
  --public-access 0.0.0.0    # Azure-services sentinel (NOT 0.0.0.0/0); use VNet/Private Link for prod

# Create the eddytor database (flag is --name / -n, NOT --database-name).
az postgres flexible-server db create \
  --resource-group "$RG" --server-name eddytor-pg --name eddytor

# Allow-list the extensions the migrations need — Azure blocks all extensions by
# default, so without this migration 0 dies with "extension citext is not
# allow-listed". Dynamic parameter, no restart needed.
az postgres flexible-server parameter set \
  --resource-group "$RG" --server-name eddytor-pg \
  --name azure.extensions --value citext,pgcrypto
```

Azure requires TLS, so the conn string **must** end with `?sslmode=require`, and the host
is the FQDN Azure prints (`eddytor-pg.postgres.database.azure.com`). Put it in the secret
that eddytor-deploy-helm creates (reusing `$PGADMIN_PW` so the password never appears as a
literal in history). **Do not enable `waitForDb`** for this external DB (see Gotchas):

```bash
kubectl -n eddytor create secret generic eddytor-secrets \
  --from-literal=EDDYTOR_DATABASE_URL="postgres://eddytor:${PGADMIN_PW}@eddytor-pg.postgres.database.azure.com:5432/eddytor?sslmode=require" \
  --from-literal=EDDYTOR_ENCRYPTION_KEY="$(openssl rand -base64 32)" \
  --from-literal=EDDYTOR_API_KEY_SECRET="$(openssl rand -base64 32)"
unset PGADMIN_PW
```

For production prefer **Microsoft Entra (Azure AD) auth** over a static admin password.

### Object store — Azure Blob (after install)

The user needs object storage to create tables. Register Azure Blob as a storage
connection after install — prefer **managed identity** (`use_msi`), or a service principal.
See **eddytor-storage-registration** for the `register_az_storage` parameters (do not
restate them here). For evaluation only, `--set garage.bundled=true` gives an in-cluster S3.

### Raw public IP, no DNS (two-step)

The LB IP isn't known until Azure provisions it, so install first, then reconcile the URL:

```bash
# 1. install with the server on a LoadBalancer (per eddytor-deploy-helm Install).
#    External DB → leave waitForDb off (chart bug, see Gotchas).
helm upgrade --install eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set secrets.existingSecret=eddytor-secrets \
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
* **`waitForDb` is broken with an external DB.** The chart's `wait-for-db` init container
  hardcodes its probe to host `eddytor-postgres` (the *bundled* Service name) regardless of
  `EDDYTOR_DATABASE_URL`, so with managed Postgres it loops forever (`nc: bad address
  'eddytor-postgres'`) and the pods stay `Init`. **Leave `waitForDb.enabled=false` (the
  default) for any external DB** — the server does its own preflight + advisory-locked
  migration, so the gate adds nothing. Only use `waitForDb` with bundled Postgres.
* **Azure blocks Postgres extensions by default.** The migrations need `citext` + `pgcrypto`;
  without allow-listing them (`az postgres flexible-server parameter set --name
  azure.extensions --value citext,pgcrypto`, see Managed Postgres) migration 0 fails with
  `extension "citext" is not allow-listed`. Set it before first server boot.
* **Azure Postgres needs `?sslmode=require`** in the conn string, and the AKS egress IP
  (or VNet) must be allowed through the server firewall, or the server can't connect.
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
