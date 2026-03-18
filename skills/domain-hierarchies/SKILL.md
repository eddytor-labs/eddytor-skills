```yaml
name: eddytor-domain-hierarchies
description: >
  Use this skill when the user wants to set up allowed value lists, create
  category-subcategory relationships, link columns across tables via reference
  domains, or manage hierarchical master data structures in Eddytor. Also use
  when the user mentions "allowed values", "enum", "dropdown", "hierarchy",
  "parent-child", "lookup", "foreign key", or "domain constraint" — even if
  they don't explicitly say "domain."
```

# Domain Hierarchies

Domains restrict which values a column accepts. Enforced on every `insert_rows` and `merge_rows`.

## Setting up a hierarchy (step by step)

This is the recommended procedure for a category → subcategory → product_type chain:

**Step 1 — Top level: fixed domain**

```
set_column_domain(table, "category", "fixed", values=["Electronics", "Clothing", "Food"])
```

**Step 2 — Second level: hierarchical domain**

```
set_column_domain(table, "subcategory", "hierarchical",
  parent_column="category",
  hierarchy={ "Electronics": ["Phones", "Laptops"], "Clothing": ["Shirts", "Pants"], ... })
```

**Step 3 — Third level: another hierarchical domain**

```
set_column_domain(table, "product_type", "hierarchical",
  parent_column="subcategory",
  hierarchy={ "Phones": ["Smartphone", "Feature Phone"], "Laptops": ["Ultrabook", "Gaming"], ... })
```

Always set parent domains before child domains.

## Reference domains (cross-table)

Link a column to another table's domain:

```
set_column_domain(table, "customer_id", "reference",
  source_table="eddytor.cfg_xxx.abc123_customers", source_column="customer_id")
```

The source column **must already have a domain configured**.

## Inspecting domains

* `get_allowed_values(table, "status")` → flat list for fixed domains
* `get_allowed_values(table, "subcategory", parent_value="Electronics")` → children for hierarchical
* `delete_column_domain(table, column, force=true)` → removes constraint

## Validation loop after setting domains

1. `validate_domain_values(table, columns=["status", "category"])` → find mismatches
2. Review suggestions (e.g., `"Electroncis" → "Electronics"`)
3. Fix with `merge_rows` → re-validate

## Gotchas

* Parent column's domain must be set **before** creating a hierarchical child domain.
* If a parent value changes, the child must be valid for the new parent — otherwise the merge is rejected.
* The `hierarchy` parameter in `set_column_domain` **replaces** the entire mapping — include all parent-child pairs, not just new ones.
* Reference domains validate **live** against the source table. Deleting source values orphans dependent rows.
* Domain values are **case-sensitive**: `"Active"` ≠ `"active"`.
* `delete_column_domain` without `force: true` fails if values are in use.
