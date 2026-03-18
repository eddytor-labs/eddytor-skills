---
name: eddytor-bulk-operations
description: >
  Inserts, updates, deletes, and upserts multiple rows atomically in Eddytor
  using merge_rows. Activates for bulk insert, batch update, upsert, merge rows,
  mass delete, data sync, or any multi-row operation — even without the word
  "bulk."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Bulk Operations

## Default tool: `merge_rows`

Use `merge_rows` for most mutations. It handles INSERT, UPDATE, and DELETE in one atomic call:

```json
{
  "table": "eddytor.cfg_xxx.abc123_products",
  "comment": "Q1 price update and product retirement",
  "rows": [
    { "_operation": "INSERT", "product_id": "P200", "name": "New Item", "category": "Clothing", "price": 19.99, "status": "active" },
    { "_operation": "UPDATE", "product_id": "P001", "price": 34.99 },
    { "_operation": "DELETE", "product_id": "P050" }
  ]
}
```

**UPDATE rows only need PK + changed fields.** Omitted columns keep existing values.

Use `insert_rows` only when all rows are new and you want a simpler call. Use `delete_rows` for PK-only deletes.

## Procedure: data sync workflow

1. `query_rows` → understand current state
2. Compute diff → which rows need INSERT, UPDATE, DELETE
3. `merge_rows` with `comment` → single atomic operation, one Delta version
4. `validate_constraints` + `validate_domain_values` → confirm quality
5. If violations: fix with another `merge_rows` → re-validate

## Constraint enforcement

All bulk operations enforce: PK uniqueness, NOT NULL, check constraints, domain constraints. If **any row** fails, the **entire batch** is rejected. Fix violating rows and retry the whole batch.

## Gotchas

* `_operation` is required on every row in `merge_rows`. Forgetting it causes an error.
* UPDATE rows: only send PK + changed fields. Sending all columns wastes bandwidth and risks overwriting concurrent changes.
* DELETE rows: only PK columns needed. Other columns are ignored.
* One `merge_rows` with 1000 rows = 1 Delta version. 1000 individual `insert_rows` = 1000 versions. Prefer fewer, larger operations.
* If MCP message size limit is reached (~5MB JSON), split into batches.
* Entire batch is atomic — partial success is not possible. This is intentional.

## Validation loop

After every bulk mutation:

- [ ] `validate_constraints` → zero violations
- [ ] `validate_domain_values` → zero mismatches
- [ ] `get_table_history` → verify expected metrics (rows inserted/updated/deleted)
