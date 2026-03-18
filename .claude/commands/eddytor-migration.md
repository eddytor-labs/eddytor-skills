Use this skill when helping the user migrate data into Eddytor from external systems — Excel, Microsoft MDS, SQL databases, or other MDM tools.

# Migration to Eddytor

## Default procedure (applies to all sources)

1. Export source data as CSV (one file per entity/sheet)
2. `infer_schema` → review types and domain candidates
3. Import reference/lookup tables **first** (e.g., Categories, Countries)
4. Set domains on reference tables
5. Import dependent tables → set reference domains pointing to step 3 tables
6. `validate_constraints` + `validate_domain_values` on every table
7. `profile_table` → verify row counts match source

Always import bottom-up: reference tables → main entities → cross-references.

## From Excel

Export each sheet as CSV, then follow the default procedure.

### Excel gotchas
* Merged cells: unmerge before exporting — merged values appear only in first row
* `"001234"` becomes `1234` as numeric — ensure `Utf8` type for IDs with leading zeros
* Standardize dates to ISO format before export — mixed formats cause type inference issues
* Formulas: "Paste as Values" before export
* Multiple sheets = multiple tables — plan names and relationships upfront

## From Microsoft MDS

| MDS | Eddytor |
| --- | --- |
| Entity | Table |
| Attribute | Column |
| Domain-based attribute | Reference domain |
| Business rule | Check constraint + domain |
| Code | Primary key |
| Hierarchy | Hierarchical domain |

### MDS procedure
1. Export entities as CSV from MDS web UI or staging tables
2. Map attribute types: `Text` → `Utf8`, `Number` → `Int64`/`Float64`, `DateTime` → `Timestamp`
3. Import reference entities first → set domains
4. Import dependent entities → set reference domains
5. Translate business rules to check constraints
6. Full validation pass

## From SQL databases

Export to CSV via database tools. Tips:
* Include headers, use ISO date format, UTF-8 encoding
* Handle NULLs explicitly
* For large tables, export in batches by PK range

## Post-migration checklist

- All tables imported with correct schemas
- PKs set and unique
- Domains configured on categorical columns
- Check constraints replicate source business rules
- `validate_constraints` passes (zero violations) on all tables
- `validate_domain_values` passes (zero mismatches) on all tables
- `profile_table` row counts match source
- Reference domains link dependent → reference tables correctly
