---
name: eddytor-deploy-eks
description: >-
  Deploys self-hosted Eddytor on Amazon EKS — the AWS-specific deltas on top of
  the cluster-agnostic Helm skill. Activates for EKS, Elastic Kubernetes Service,
  eksctl, deploy on AWS, RDS for Postgres, S3 object store, ALB/NLB ingress, or
  IRSA — even without the phrase "deploy on EKS."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Deploy Eddytor on Amazon EKS

EKS-specific layer over **eddytor-deploy-helm**. That base skill owns the chart
install, the three required secrets, the verify step, and first-admin setup — read
it for those. This skill only covers what differs on AWS: cluster creation, RDS,
S3 via IRSA, and ALB/NLB ingress. Install from the OCI chart
(`oci://ghcr.io/nordalf/charts/eddytor`), never clone or build.

## Gather inputs first (prompt the user — never assume)

Before creating anything, **ask the operator** (use AskUserQuestion) and use their
answers in every command. The names below are examples, not defaults. Confirm:

- **AWS account / profile** (`aws sts get-caller-identity`) and **region**.
- **Cluster name** (example `eddytor`).
- **Node type + count** (and managed vs self-managed node group).
- **Networking — which VPC, if any:**
  - *Let eksctl create a new VPC* (default, simplest), or
  - *Use existing subnets:* pass `--vpc-private-subnets`/`--vpc-public-subnets` with
    subnet IDs (or a ClusterConfig file). Required to land in an existing VPC / shared
    networking, and to reach an RDS instance in that VPC.
  - *Endpoint access:* public, private, or both (`--vpc-private-access` / public access CIDRs).
- **Kubernetes namespace** (example `eddytor`).

Only proceed once these are settled — the operator owns what gets created.

## Default procedure

1. Create the cluster (`eksctl create cluster`) with the OIDC provider enabled for IRSA.
2. `aws eks update-kubeconfig` so `kubectl`/`helm` target the new cluster.
3. Provision **RDS for PostgreSQL**; allow the node group's traffic in its security group.
4. `kubectl create namespace eddytor` + create the secret (base skill) with the RDS URL.
5. `helm upgrade --install` with `postgres.bundled=false`, `waitForDb.enabled=true`,
   and your ingress choice (ALB or NLB).
6. Verify the DB handshake + migrations and provision the first admin (base skill).
7. Register S3 storage via IRSA (cross-reference **eddytor-storage-registration**).

## Tool usage examples

Create the cluster (provisions VPC + a managed node group) and enable IRSA:

```bash
# Substitute the name/region/node type/network the operator chose above.
eksctl create cluster --name "$CLUSTER" --region "$REGION" \
  --nodes "$NODES" --node-type "$NODE_TYPE" --with-oidc
  # existing VPC: --vpc-private-subnets subnet-aaa,subnet-bbb --vpc-public-subnets subnet-ccc,subnet-ddd
aws eks update-kubeconfig --name "$CLUSTER" --region "$REGION"
```

`--with-oidc` associates the IAM OIDC provider — required for IRSA. On an
existing cluster: `eksctl utils associate-iam-oidc-provider --cluster eddytor --approve`.

RDS connection string in the secret — note the **required** SSL mode:

```bash
kubectl -n eddytor create secret generic eddytor-secrets \
  --from-literal=EDDYTOR_DATABASE_URL="postgres://eddytor:PASS@eddytor.abc123.eu-west-1.rds.amazonaws.com:5432/eddytor?sslmode=require" \
  --from-literal=EDDYTOR_ENCRYPTION_KEY="$(openssl rand -base64 32)" \
  --from-literal=EDDYTOR_API_KEY_SECRET="$(openssl rand -base64 32)"
```

Install against RDS (external Postgres) with ALB ingress + an ACM cert:

```bash
helm upgrade --install eddytor oci://ghcr.io/nordalf/charts/eddytor -n eddytor \
  --set secrets.existingSecret=eddytor-secrets \
  --set postgres.bundled=false \
  --set waitForDb.enabled=true \
  --set config.publicUrl="https://eddytor.example.com" \
  --set ingress.enabled=true \
  --set ingress.host=eddytor.example.com \
  --set ingress.className=alb \
  --set ingress.tls=true
```

The AWS Load Balancer Controller (ALB ingress) and ACM cert are AWS prerequisites
configured outside the chart — install/annotate per AWS docs. `ingress.className=alb`
points the chart's Ingress at that controller.

S3 via IRSA — create the IAM role, then bind it to the Eddytor service account so
S3 is reached **without static access keys**:

```bash
eksctl create iamserviceaccount --cluster eddytor --namespace eddytor \
  --name eddytor-server --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --approve --override-existing-serviceaccounts
```

`AmazonS3FullAccess` is shown for brevity — it is **over-broad for production**. Scope a
custom policy to the specific bucket(s) Eddytor uses (`s3:GetObject`/`PutObject`/
`DeleteObject`/`ListBucket` on `arn:aws:s3:::your-bucket[/*]`) and attach that instead.

Then register the bucket with **register_s3_storage** (no keys needed — the pod
assumes the role). See **eddytor-storage-registration** for that tool's parameters.

## Gotchas

* **`?sslmode=require` on the RDS URL** — RDS rejects/encourages-against plaintext;
  omit it and the handshake can fail or warn. Always set it.
* **RDS security group must allow the node group.** If the server pod sticks in
  `Init` (waitForDb can't reach Postgres), the inbound 5432 rule for the node-group
  SG is the usual cause — not a chart problem.
* **IRSA needs the OIDC provider.** Without `--with-oidc` (or the associate step),
  the service-account role annotation does nothing and S3 calls 403. Confirm with
  `kubectl -n eddytor get sa eddytor-server -o yaml` (must show the role-arn annotation).
* **Bundled Postgres can crash-loop on the EBS CSI** with
  `chown: Operation not permitted` — the chart drops all Linux caps and the EBS
  volume root is root-owned. Either use RDS (recommended), or patch the StatefulSet
  to add `CHOWN,FOWNER,DAC_OVERRIDE,SETGID,SETUID`. Local hostPath clusters don't hit this.
* **NLB endpoint is unknown until provisioned.** `--set server.service.type=LoadBalancer`
  gives a raw NLB hostname with no DNS/TLS; publicUrl can't be set up front. Use the
  two-step from the base skill: install, read the assigned hostname, then
  `helm upgrade --reuse-values --set config.publicUrl=http://<nlb-host>:8080`.
* **Use the `:k8s` images (chart default)** and **one release per namespace** — the
  engine Services are fixed-named (`eddytor-engine` / `eddytor-engine-headless`).

## Guidelines

* Prefer **RDS + S3-via-IRSA** for anything you keep; bundled Postgres/Garage are
  evaluation-only and HA-less.
* Pick **one** ingress path: ALB (`ingress.className=alb` + ACM, gives DNS/TLS) for
  production, or NLB (`server.service.type=LoadBalancer`) for a quick raw endpoint.
* If the UI is enabled, expose it on its own origin and set `ui.origin` (base skill).
* Teardown removes the cluster, VPC, and node group in one shot:
  `eksctl delete cluster --name eddytor --region eu-west-1`. RDS instances created
  outside eksctl are deleted separately.
* All chart-install, secret, verify, and first-admin specifics live in
  **eddytor-deploy-helm** — defer to it rather than restating.
