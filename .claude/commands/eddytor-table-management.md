Use this skill when helping the user create tables, add/modify columns, change primary keys or constraints, rename or move tables, or understand schemas and data types in Eddytor.

# Table Management

## Creating a table

Use `create_table`. At least one column must have `is_primary_key: true`.

```json
{
  "table_name": "products",
  "location": "abfss://mdm@storage.dfs.core.windows.net/master-data",
  "description": "Product master data",
  "columns": [
    { "name": "product_id", "data_type": "Utf8", "nullable": false, "is_primary_key": true },
    { "name": "name", "data_type": "Utf8", "nullable": false },
    { "name": "category", "data_type": "Utf8", "nullable": false },
    { "name": "price", "data_type": "Float64", "nullable": true },
    { "name": "status", "data_type": "Utf8", "nullable": false }
  ],
  "constraints": [{ "name": "price_positive", "expr": "price > 0" }]
}
```

Supported data types: `Utf8`, `Int32`, `Int64`, `Float64`, `Boolean`, `Date32`, `Timestamp`

## Procedure: schema introspection before any change

1. `get_table_schema` — columns, types, nullability
2. `get_table_metadata` — constraints, PKs, version, description
3. Only then proceed with modifications

## Adding columns

`add_column` — existing rows get NULL for new columns:

```json
{ "table": "eddytor.cfg_xxx.abc123_products", "columns": [
    { "name": "weight_kg", "data_type": "Float64", "nullable": true }
]}
```

## Modifying column settings

`update_column_settings` toggles `is_primary_key`, `is_unique`, `is_nullable`.
Enabling uniqueness or PK validates existing data first. Disabling constraints is always safe.

## Renaming and moving

* `rename_table` — new path, same storage config. `destination_path` is relative (e.g., `europe/products`), never a URL.
* `move_table` — different storage config. Requires `destination_config_id` (from `list_tables`).

## Gotchas

* `table_name` in `create_table` is **bare only** — never `catalog.schema.products`.
* `location` is a **full URL** — not a relative path.
* Column names are case-sensitive. Be consistent with source system conventions.
* `Float64` for prices, not `Float32` — precision matters for MDM.
* Use `Utf8` for IDs unless certain they're always numeric — preserves leading zeros.
* `update_column_settings` with `is_nullable: false` checks for existing NULLs and rejects if found.

## Validation loop

After any schema change:
1. `get_table_schema` to confirm change applied
2. `validate_constraints` to ensure existing data still passes
3. If failures: fix data with `merge_rows`, re-validate
