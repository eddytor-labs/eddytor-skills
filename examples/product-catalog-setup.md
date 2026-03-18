# Example: Product catalog setup

End-to-end walkthrough: create a product master data table, import data, set domains, validate.

## 1. Create the table

```
create_table(
  table_name="products",
  location="abfss://mdm@storage.dfs.core.windows.net/master-data",
  description="Product master data catalog",
  columns=[
    { name: "product_id", data_type: "Utf8", nullable: false, is_primary_key: true },
    { name: "name", data_type: "Utf8", nullable: false },
    { name: "category", data_type: "Utf8", nullable: false },
    { name: "subcategory", data_type: "Utf8", nullable: false },
    { name: "price", data_type: "Float64", nullable: true },
    { name: "status", data_type: "Utf8", nullable: false },
    { name: "supplier_code", data_type: "Utf8", nullable: true }
  ],
  constraints=[{ name: "price_positive", expr: "price > 0" }]
)
```

## 2. Set domain constraints

```
# Status: fixed domain
set_column_domain(table, "status", "fixed", values=["active", "discontinued", "draft"])

# Category: fixed domain
set_column_domain(table, "category", "fixed",
  values=["Electronics", "Clothing", "Home & Garden", "Food & Beverage"])

# Subcategory: hierarchical domain linked to category
set_column_domain(table, "subcategory", "hierarchical",
  parent_column="category",
  hierarchy={
    "Electronics": ["Phones", "Laptops", "Accessories"],
    "Clothing": ["Shirts", "Pants", "Outerwear"],
    "Home & Garden": ["Furniture", "Tools", "Lighting"],
    "Food & Beverage": ["Snacks", "Drinks", "Fresh"]
  })
```

## 3. Insert initial data

```
merge_rows(table, comment="Initial product catalog load", rows=[
  { _operation: "INSERT", product_id: "P001", name: "Smartphone X", category: "Electronics", subcategory: "Phones", price: 699.99, status: "active", supplier_code: "SUP-001" },
  { _operation: "INSERT", product_id: "P002", name: "Cotton Tee", category: "Clothing", subcategory: "Shirts", price: 24.99, status: "active", supplier_code: "SUP-002" },
  { _operation: "INSERT", product_id: "P003", name: "Standing Desk", category: "Home & Garden", subcategory: "Furniture", price: 449.00, status: "draft", supplier_code: "SUP-003" }
])
```

## 4. Validate

```
validate_constraints(table)     # Check price > 0, NOT NULL, PK uniqueness
validate_domain_values(table)   # Check status, category, subcategory values
profile_table(table)            # Verify 3 rows, check distributions
```

## 5. Verify with a query

```
query_rows(table, columns=["product_id", "name", "category", "price"], filter="status = 'active'", order_by="price DESC")
```

Expected result: P001 (699.99), P002 (24.99).

## Key takeaways

- Set domains **before** inserting data — catches errors at write time
- Use `merge_rows` with `comment` for traceability
- Always validate after mutations
- Use `query_rows` (not `execute_sql`) for simple reads
