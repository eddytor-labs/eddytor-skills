```yaml
name: eddytor-data-quality
description: >
  Use this skill when the user wants to enforce data quality rules, set allowed
  values on columns, validate constraints, profile data distributions, find
  domain violations, or clean dirty data in Eddytor. Also use when the user
  mentions "data quality", "validation", "allowed values", "constraints",
  "profiling", "domain", or asks about anomaly detection — even if they
  don't explicitly say "data quality."
```

# Data Quality

Eddytor enforces quality through: **domain constraints** (restrict allowed values), **check constraints** (SQL expressions that must hold), and **validation tools** (scan for violations after the fact).

## Setting domain constraints

Use `set_column_domain`. Three types — pick the one that fits:

**Fixed** (enum): `domain_type: "fixed"`, `values: ["active", "inactive", "draft"]`
Use for: status codes, country codes, any finite value set.

**Hierarchical** (parent-child): `domain_type: "hierarchical"`, `parent_column: "category"`, `hierarchy: { "Electronics": ["Phones", "Laptops"], ... }`
Use for: category → subcategory, country → region.

**Reference** (foreign key): `domain_type: "reference"`, `source_table: "eddytor.cfg_xxx.abc123_customers"`, `source_column: "customer_id"`
Use for: cross-table referential integrity. Source column must already have a domain.

For detailed hierarchy setup patterns, see the domain-hierarchies skill.

## Validation procedure (run after every bulk mutation)

1. `validate_constraints` → check expressions. Returns violations with sample rows.
2. `validate_domain_values` → scan for mismatches. Returns typo suggestions (e.g., `"actve" → "active"`).
3. If violations found: fix with `merge_rows` → re-run steps 1-2.
4. `profile_table` → final sanity check on row count, null distribution, value ranges.

## Gotchas

* Set domains **before** importing data. Rejecting at write time is cheaper than cleaning up after.
* Domain values are **case-sensitive** — `"Active"` and `"active"` are different values.
* Hierarchical domains: set the parent column's domain **first**, then the child.
* Reference domains are **live** — deleting a value from the source table can orphan references in dependent tables. Use `validate_domain_values` to find orphans.
* `delete_column_domain` with `force: true` removes the constraint regardless of existing data — use with caution.
* `validate_domain_values` returns similarity suggestions but doesn't auto-fix. Use `merge_rows` to apply corrections.

## Identifying domain candidates

Use `profile_table` to check distinct counts. If a string column has <20 distinct values in 10,000+ rows, it's a good domain candidate. `infer_schema` also flags low-cardinality columns automatically during import.

## Guidelines

* Always validate after bulk operations — `validate_constraints` + `validate_domain_values` as a pair
* Use `get_allowed_values` to inspect existing domains before modifying them
* Profile before setting domains — understand cardinality first
