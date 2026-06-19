---
name: eddytor-table-lifecycle
description: >
  Renames, moves, and permanently deletes Eddytor tables, and resolves or shares
  them. Activates for rename table, move table to another storage, relocate,
  delete/drop a table, share link, or resolving a table by id — even without
  "lifecycle."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Table Lifecycle

Rename, relocate, share, and delete existing tables. For creating tables and editing
their schema, see eddytor-table-management.

## Default procedure

1. `list_tables` → get the fully-qualified `table` name, its `storage_path`, and
   `config_id`.
2. Apply the lifecycle operation below.
3. Re-run `list_tables` to confirm the new path/identity (rename/move change the path).

## Tool usage examples

```
# Rename: new path within the SAME storage config. destination_path is RELATIVE, never a URL.
rename_table(table="eddytor.cfg_xxx.abc123_products", destination_path="europe/products")

# Move: to a DIFFERENT storage config. Needs the target config_id (from list_tables).
move_table(table="eddytor.cfg_xxx.abc123_products",
  destination_config_id="550e8400-e29b-41d4-a716-446655440000",
  destination_path="master-data/products")

# Resolve a table's location/identity by its table_id
resolve_table(table_id="abc123...")

# Get a shareable link
get_share_link(table="eddytor.cfg_xxx.abc123_products")

# Permanently delete — destructive, irreversible. confirm MUST be true.
drop_table(table="eddytor.cfg_xxx.abc123_products", confirm=true)
```

## Gotchas

* `destination_path` for both rename and move is a **relative** path (e.g.
  `europe/products`), never a URL like `abfss://...` or `s3://...`.
* Rename, move, and drop are all **refused while another table references this one** via
  a domain (cross-table reference). Remove the dependent reference domain first.
* `drop_table` is **irreversible** — it deletes all data and Delta-log files, so there is
  no time-travel or rollback afterward. It requires the **Builder** or **Admin** role and
  `confirm=true` (omitting/`false` is rejected as a safety guard).
* Rename/move copy the underlying `_delta_log` and data files, so the table keeps its
  identity (the stable `table_id`) but its fully-qualified name/path changes — refresh it
  from `list_tables` before further calls.

## Guidelines

* Prefer `rename_table` for in-place reorganisation; reserve `move_table` for crossing
  storage configs (dev → prod).
* Before `drop_table`, consider `get_table_history` / a final `query_rows` export — there
  is no undo.
* To move raw objects rather than a registered table, use `move_objects` (see
  eddytor-object-store).
