Use this skill when helping the user connect to Eddytor for the first time, set up MCP, authenticate, or create their first table.

# Getting Started with Eddytor

Eddytor is an MDM platform. Interact with it through MCP tools: create tables, import data, enforce constraints, manage master data lifecycle.

## Connecting

MCP endpoint: `https://mcp.eddytor.com/mcp`
Auth: OAuth 2.1 (Supabase-backed), handled automatically by MCP clients.

## First interaction

1. Call `list_tables` — see all tables with fully-qualified names, storage paths, last modified
2. Pick a table → `get_table_schema` for columns/types → `get_table_metadata` for constraints/PKs/version
3. Read data with `query_rows` (default) or `execute_sql` (joins/CTEs only)

## Gotchas

* Every tool that takes a `table` parameter requires the **fully-qualified name** (`catalog.schema.table`). Get this from `list_tables` — never construct it manually.
* `table_name` in `create_table` and `import_csv` is the **bare name only** (e.g., `products`), never prefixed.
* `location` is a **full storage URL** (e.g., `abfss://container@account.dfs.core.windows.net/path`).
* Table names in `execute_sql` must be backtick-quoted per component: `` `catalog`.`schema`.`table` ``. Unquoted dots cause parse errors.
* Every table needs at least one primary key column.

## Common first-session workflows

**Import existing data:**
1. `infer_schema` with CSV content → review types and domain candidates
2. `import_csv` → set domains on flagged columns → `validate_domain_values`

**Create from scratch:**
1. `create_table` with column defs (at least one PK)
2. `set_column_domain` on categorical columns
3. `insert_rows`

**Explore existing data:**
1. `list_tables` → `get_table_schema` → `profile_table` → `query_rows` with `limit: 10`

## Guidelines

* Always call `list_tables` first — never assume table names
* Default to `query_rows` over `execute_sql`
* Check `get_table_history` before destructive operations
* Set domain constraints early, before importing data
