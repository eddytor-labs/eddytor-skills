---
name: eddytor-object-store
description: >
  Browses and manages raw files in a registered Eddytor storage configuration —
  list, upload, download, delete, organise into folders, and move objects between
  configs. Activates for listing files, uploading/downloading objects, folders,
  moving data between buckets, or seeding a demo table — even without "object store."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Object Store

These tools operate on the raw objects in a storage configuration (the bucket/container
behind a `config_id`), independent of Delta tables. Get the `config_id` from
`list_storage_configs` (see eddytor-storage-registration).

## Default procedure

1. `list_storage_configs` → pick a `config_id`.
2. `list_objects(config_id, delimiter=true)` → browse folders and files.
3. Upload/download/delete/move as needed with the tools below.

## Tool usage examples

```
# Browse: delimiter=true gives directory-style listing (folders in common_prefixes)
list_objects(config_id, path="master-data", delimiter=true, extensions="csv,parquet", limit=100)
# paginate with the returned continuation_token

# Download (returns { path, filename, size_bytes, content_base64 }); max 8 MiB
download_object(config_id, path="master-data/products.csv")

# Upload from base64; path = destination directory, filename = object name. Max 100 MiB
upload_object(config_id, path="master-data", filename="products.csv", content_base64="<...>")

# Organise
create_folder(config_id, path="master-data/archive")
delete_object(config_id, path="master-data/old.csv")

# Move every object under a prefix (optionally to a different config)
move_objects(source_config_id, source_path="staging/products",
  destination_config_id, destination_path="master-data/products")

# Seed a demo table to explore domains
create_demo_table(config_id)        # creates demo_products at the config root
```

## Gotchas

* `list_objects` `limit` defaults to 100, max 1000; use `continuation_token` to page.
  `delimiter=true` returns folders separately in `common_prefixes`.
* `download_object` rejects objects over **8 MiB** — use the REST download endpoint for
  larger files. `upload_object` caps decoded content at **100 MiB**.
* `upload_object` `filename` must have no path separators; the destination is
  `path/filename`.
* `delete_object` targets a single object, not a prefix. To move/remove a whole prefix
  use `move_objects`.
* `move_objects` deregisters any Delta table at the source and re-runs discovery at the
  destination — paths shift, so re-check `list_tables` afterward. To relocate a *table*
  specifically (not raw objects), prefer `move_table` (see eddytor-table-lifecycle).
* Paths are validated: no `..`, no leading `/`, no `//`, no null bytes.

## Guidelines

* Use `extensions` to filter noisy listings (e.g. `"csv"`).
* For tabular data you intend to query, import it (`import_csv`) rather than leaving it
  as a raw object — raw files are read-only even when discovered.
* `create_demo_table` is the fastest way to get a constrained table to experiment with.
