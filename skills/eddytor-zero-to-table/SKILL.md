---
name: eddytor-zero-to-table
description: >
  End-to-end guided path from an empty Eddytor instance to a constrained, populated,
  validated table. Activates for "set up from scratch", first table, onboarding a new
  dataset, "I have a CSV and nothing else", or "get me started" — even without naming
  a specific tool.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Zero to Table

The full sequence for standing up master data from nothing: connect storage, land the
data, constrain it, validate it. Each step links to the deep-dive skill.

## Default procedure

**1. Connect a storage configuration** (eddytor-storage-registration)

```
register_s3_storage(bucket_name="mdm-prod", region="eu-west-1",
  access_key_id="...", secret_key="...")
list_storage_configs()              # capture the config_id
```

Nothing can be created until a storage config exists.

**2. Land the data** — pick one:

* *Have a CSV?* Infer then import (eddytor-data-import):

  ```
  infer_schema(csv_content="id,name,category\n...")   # review types + domain candidates
  import_csv(table_name="products",
    location="abfss://mdm@acme.dfs.core.windows.net/master-data",
    primary_key_column="product_id",
    csv_content="...", non_nullable_columns=["product_id","name"])
  ```

* *Designing from scratch?* Create the table (eddytor-table-management):

  ```
  create_table(table_name="products", location="...",
    columns=[{ name:"product_id", data_type:"Utf8", nullable:false, is_primary_key:true }, ...],
    constraints=[{ name:"price_positive", expr:"price > 0" }])
  ```

`table_name` is bare (`products`); `location` is a full storage URL.

**3. Constrain it** (eddytor-data-quality, eddytor-domain-hierarchies)

```
set_column_domain(table, "status", "fixed", values=["active","discontinued","draft"])
# hierarchical domains are keyed by the parent value's UUID — read it first:
get_allowed_values(table, "category")
set_column_domain(table, "subcategory", "hierarchical",
  parent_column="category", hierarchy={ "<parent-uuid>": ["Phones","Laptops"] })
```

**4. Validate** (eddytor-data-quality)

```
validate_constraints(table)       # PK, NOT NULL, check expressions
validate_domain_values(table)     # domain mismatches + typo suggestions
profile_table(table)              # row count, null distribution, ranges
```

Fix any violations with `merge_rows`, then re-validate.

## Gotchas

* Set domains **before** loading data when you can — write-time rejection beats cleanup.
* If `import_csv` fails (type conflict, duplicate PKs), the table is **not created** — fix
  the data and retry.
* Hierarchical `hierarchy` maps use parent-value **UUIDs**, not strings (step 3).
* Get the fully-qualified `table` name from `list_tables` for every step after creation.

## Guidelines

* This skill is the map; defer to the linked skill for each step's full options.
* Confirm each step before the next: `get_table_schema` after create/import,
  `get_allowed_values` after setting a domain, validation before declaring done.
