Use this skill when helping the user read, filter, sort, aggregate, or analyze data in Eddytor tables.

# Querying Data

## Tool selection: default to `query_rows`

| Need | Tool |
| --- | --- |
| Read with filters, sort, paginate | `query_rows` (default) |
| Count, sum, avg, min, max with grouping | `aggregate` |
| Joins, subqueries, CTEs, window functions | `execute_sql` |

Use `query_rows` unless you need something it can't express.

## `query_rows`

```json
{
  "table": "eddytor.cfg_xxx.abc123_products",
  "columns": ["product_id", "name", "price"],
  "filter": "status = 'active' AND price > 50",
  "order_by": "price DESC",
  "limit": 20
}
```

Filter syntax: standard SQL WHERE (`=`, `!=`, `LIKE`, `IN (...)`, `IS NULL`, `BETWEEN`).
Order syntax: `"price DESC"` or `"category ASC, price DESC"`.

## `aggregate`

```json
{
  "table": "eddytor.cfg_xxx.abc123_products",
  "aggregations": ["COUNT(*)", "AVG(price)"],
  "group_by": ["category"],
  "filter": "status != 'deleted'"
}
```

## `execute_sql` — only when needed

Table names **must** be backtick-quoted per component:

```sql
SELECT p.`category`, COUNT(*) as cnt
FROM `eddytor`.`cfg_xxx`.`abc123_products` p
WHERE p.`status` = 'active'
GROUP BY p.`category`
HAVING COUNT(*) > 5
```

## Gotchas

* Unquoted dots in `execute_sql` table names cause parse errors. Always backtick-quote each component.
* String comparisons need quotes in filters: `"name = 'Widget'"`. Numeric don't: `"price > 50"`.
* NULL comparisons: use `IS NULL` / `IS NOT NULL`, never `= NULL`.
* Date filtering uses ISO format: `"order_date > '2024-01-15'"`.
* `aggregate` operates on rows within one table. It cannot count tables or aggregate across tables.

## Procedure: exploring unfamiliar data

1. `get_table_schema` → understand columns and types
2. `query_rows` with `limit: 10` → see sample data
3. `profile_table` → statistical overview (nulls, distinct counts, ranges)
4. Refine queries based on what you learned

## Guidelines

* Select only needed columns — faster than fetching everything
* Set `limit: 100` by default — increase only when the user needs more
* Use filters to push work to the engine, not client-side
