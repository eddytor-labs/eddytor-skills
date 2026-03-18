# Example: Data cleanup workflow

Import a messy CSV, identify quality issues, fix them, validate.

## 1. Preview the data

```
infer_schema(csv_content="id,name,category,status,price\n001,Widget A,electronics,Active,29.99\n002,Gadget B,Electronics,active,15.50\n003,Thing C,electroncis,ACTIVE,0\n004,,Electronics,active,-5\n005,Item E,Clothing,,12.99")
```

Schema inference flags:
- `category` and `status` as domain candidates (low cardinality)
- `id` as `Int64` (but has leading zeros — should be `Utf8`)

## 2. Import with corrections

```
import_csv(
  table_name="raw_products",
  location="abfss://mdm@storage.dfs.core.windows.net/staging",
  primary_key_column="id",
  csv_content="...",
  non_nullable_columns=["id", "name"]
)
```

## 3. Profile and identify issues

```
profile_table(table)
```

Findings:
- `name`: 1 null (row 004)
- `category`: 3 distinct values including typo "electroncis"
- `status`: 3 distinct values with inconsistent casing ("Active", "active", "ACTIVE")
- `price`: includes 0 and -5

## 4. Set domains to define what's valid

```
set_column_domain(table, "category", "fixed", values=["Electronics", "Clothing"])
set_column_domain(table, "status", "fixed", values=["active", "discontinued", "draft"])
```

## 5. Find violations

```
validate_domain_values(table, columns=["category", "status"])
```

Returns:
- `category`: "electronics" → "Electronics", "electroncis" → "Electronics"
- `status`: "Active" → "active", "ACTIVE" → "active"

```
validate_constraints(table)
```

Returns:
- Row 004: `name` is NULL (violates NOT NULL)
- Row 003: `price` is 0, row 004: `price` is -5 (if price > 0 constraint exists)

## 6. Fix with merge_rows

```
merge_rows(table, comment="Data cleanup: fix casing, typos, missing values", rows=[
  { _operation: "UPDATE", id: "001", category: "Electronics", status: "active" },
  { _operation: "UPDATE", id: "002", status: "active" },
  { _operation: "UPDATE", id: "003", category: "Electronics", status: "active", price: null },
  { _operation: "UPDATE", id: "004", name: "Unknown Product", status: "active", price: null },
  { _operation: "UPDATE", id: "005", status: "draft" }
])
```

## 7. Re-validate

```
validate_constraints(table)      # Should pass
validate_domain_values(table)    # Should pass
profile_table(table)             # Final check
```

## Key takeaways

- Always `infer_schema` before importing — catches type issues early
- Use `profile_table` to find quality issues systematically
- `validate_domain_values` suggests corrections — apply them with `merge_rows`
- Iterate: set domains → validate → fix → re-validate until clean
