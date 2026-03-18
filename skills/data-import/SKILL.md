```yaml
name: eddytor-data-import
description: >
  Use this skill when the user wants to import CSV data into Eddytor, preview a
  schema before importing, or migrate spreadsheet data into tables. Also use when
  the user mentions "CSV import", "load data", "upload data", "infer schema", has
  raw CSV content, or asks about type detection and delimiter handling — even if
  they don't explicitly say "import."
```

# Data Import

## Default workflow: infer → import → constrain → validate

1. `infer_schema` with CSV content → review column types and domain candidates
2. `import_csv` with reviewed schema → creates table + inserts rows in one step
3. `set_column_domain` on domain candidates flagged by step 1
4. `validate_domain_values` + `profile_table` → confirm data quality

Always start with `infer_schema`. Skipping it means type surprises after import.

## `infer_schema` — preview before committing

Returns column names, Arrow types, auto-detected delimiter, and **domain candidates** (low-cardinality string columns with distinct values listed).

Review the output. Common adjustments:

* Dates as `Utf8` → should be `Date32`/`Timestamp`
* IDs as `Int64` → use `Utf8` if they have leading zeros
* Boolean-like strings (`"yes"/"no"`) → set a fixed domain after import

## `import_csv` — one-step table creation

```json
{
  "table_name": "products",
  "location": "abfss://container@account.dfs.core.windows.net/master-data",
  "primary_key_column": "product_id",
  "csv_content": "product_id,name,category,price\nP001,Widget,Electronics,29.99\n...",
  "non_nullable_columns": ["product_id", "name", "category"]
}
```

`delimiter` auto-detects `,` `;` `\t` `|` — set explicitly only if auto-detection fails.

## Post-import validation loop

1. `profile_table` → verify row count, check null distributions
2. `validate_constraints` → check expressions hold
3. `validate_domain_values` → catch typos in constrained columns
4. If issues: fix with `merge_rows` → re-validate

## Gotchas

* `table_name` is bare only (`products`), never prefixed with catalog/schema.
* `primary_key_column` must be a column name that exists in the CSV.
* If import fails (type conflict, duplicate PKs), the table is **not created** — fix data and retry.
* `csv_content` has MCP message size limits (~5MB). For larger files: create the table with a first batch, then use `insert_rows` for the rest.
* Empty strings in CSV are not the same as NULL — check nullability settings.
* `"001234"` becomes `1234` if inferred as `Int64` — ensure `Utf8` type to preserve leading zeros.
