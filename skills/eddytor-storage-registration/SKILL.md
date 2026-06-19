---
name: eddytor-storage-registration
description: >
  Connects Eddytor to an object store (S3/MinIO, Azure Blob/OneLake, or Google
  Cloud Storage) so it can discover and manage tables there. Activates for
  connecting storage, registering a bucket or container, S3/Azure/GCS setup,
  credentials, or "where do my tables live" — even without the word "storage."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Storage Registration

A **storage configuration** points Eddytor at a bucket/container and discovers the
tables inside it. You must register at least one before you can create or read any
table — `create_table`/`import_csv` write into a registered config's storage.

## Default procedure

1. Register the store with the matching `register_*` tool (below). Registration also
   runs discovery and returns the count of tables found.
2. `list_storage_configs` → grab the `id` (this is the `config_id` used everywhere else).
3. Create or import tables (see eddytor-table-management / eddytor-data-import), or
   browse raw files (see eddytor-object-store).

## Tool usage examples

**S3 / S3-compatible (MinIO)** — `bucket_name` becomes the config name. `endpoint` is
only needed for S3-compatible services:

```
register_s3_storage(
  bucket_name="mdm-prod",
  region="eu-west-1",
  access_key_id="AKIA...",
  secret_key="...",
  base_path="master-data")        # optional: scope discovery to a prefix

# MinIO / local S3-compatible:
register_s3_storage(bucket_name="eddytor", region="us-east-1",
  access_key_id="...", secret_key="...", endpoint="http://localhost:9000")
```

**Azure Blob / OneLake** — config name is `{account_name}-{container}`. Provide exactly
**one** auth mode:

```
# access key
register_az_storage(account_name="acme", container="mdm", access_key="...")

# service principal
register_az_storage(account_name="acme", container="mdm",
  client_id="...", client_secret="...", tenant_id="...")

# managed identity (only when Eddytor runs on Azure infra)
register_az_storage(account_name="acme", container="mdm", use_msi=true)

# Microsoft Fabric / OneLake
register_az_storage(account_name="acme", container="mdm",
  access_key="...", use_fabric_endpoint=true)
```

If the calling user has a linked Azure identity, omit all auth args and a bearer token
is resolved automatically.

**Google Cloud Storage** — `bucket_name` becomes the config name. Provide either the
full service-account JSON key (as a string) or an OAuth token:

```
register_gcs_storage(bucket_name="mdm-prod",
  service_account_key="{\"type\":\"service_account\", ...}")
```

**Manage configs:**

```
list_storage_configs()
# -> [{ id, name, path, scheme_type, created_at, discovery_config }]

update_storage_config(config_id, config_name="mdm-archive", discover_files=true)
delete_storage_config(config_id)    # soft-delete; object-store data is NOT deleted
```

## Discovery flags (all three register tools)

* `discover_delta` — **default true**. Registers Delta tables found under `base_path`.
* `discover_files` — default false. Registers standalone Parquet/CSV files as
  **read-only** tables.
* `discover_iceberg` — default false.
* `base_path` — scope discovery to a prefix instead of the whole bucket/container.

`update_storage_config` re-runs discovery, so flip `discover_files`/`base_path` there to
pick up more tables later.

## Gotchas

* Register storage **first** — there is nowhere to create a table until a config exists.
* Configs are per-caller. The `id` from `list_storage_configs` is the `config_id` every
  object tool and `move_table` needs.
* `delete_storage_config` is a soft delete that only deregisters tables — it never
  deletes the underlying bucket data.
* Files discovered via `discover_files` are **read-only** (no insert/merge/delete) — only
  Delta tables are writable.
* Azure: passing more than one auth mode is ambiguous — pick one.

## Guidelines

* Use a narrow `base_path` in production so discovery doesn't scan an entire bucket.
* Prefer service principal or managed identity over long-lived access keys on Azure.
* After registering, always `list_storage_configs` to capture the `config_id`.
