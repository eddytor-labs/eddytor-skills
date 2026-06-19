---
name: eddytor-mcp-tool-reference
description: >
  Provides a complete catalog of Eddytor's 45 MCP tools with parameter details
  and universal gotchas. Activates when looking up tool parameters, debugging
  failing tool calls, or listing available tools.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# MCP Tool Reference

This is a reference document — load it when you need parameter details for a specific tool. For workflow guidance, see the domain-specific skills instead.

## Quick lookup: which tool for what

**Discovery:** `list_tables`, `resolve_table`, `get_table_schema`, `get_table_metadata`, `profile_table`, `get_share_link`
**Creation & lifecycle:** `create_table`, `add_column`, `update_column_settings`, `rename_table`, `move_table`, `drop_table`
**Reading:** `query_rows` (default), `aggregate`, `execute_sql`
**Writing:** `insert_rows`, `merge_rows`, `delete_rows`
**Import:** `infer_schema`, `import_csv`
**Domains:** `set_column_domain`, `get_allowed_values`, `delete_column_domain`
**Validation:** `validate_constraints`, `validate_domain_values`
**History:** `get_table_history`, `diff_table_versions`, `rollback_table`, `restore_table`, `get_audit_log`
**Maintenance:** `optimize_table`, `vacuum_table`
**Storage configs:** `register_s3_storage`, `register_az_storage`, `register_gcs_storage`, `list_storage_configs`, `update_storage_config`, `delete_storage_config`
**Object store:** `list_objects`, `download_object`, `upload_object`, `delete_object`, `create_folder`, `move_objects`, `create_demo_table`

Total: 45 tools.

## Tool inventory

| Tool | Category | Description |
| --- | --- | --- |
| `list_tables` | Discovery | List all registered tables |
| `resolve_table` | Discovery | Resolve a table's location/identity by `table_id` |
| `get_table_schema` | Discovery | Get Arrow schema with column types and metadata |
| `get_table_metadata` | Discovery | Get table metadata (version, size, constraints) |
| `profile_table` | Discovery | Statistical profiling (nulls, cardinality, min/max) |
| `get_share_link` | Discovery | Get a shareable link for a table |
| `create_table` | DDL | Create a new Delta table |
| `add_column` | DDL | Add columns to existing table |
| `update_column_settings` | DDL | Toggle isUnique/isPrimaryKey/isNullable |
| `rename_table` | Storage | Rename table path within same config |
| `move_table` | Storage | Move table to different storage config |
| `drop_table` | DDL | Delete a table (requires `confirm: true`) |
| `query_rows` | Querying | Filtered reads with column selection |
| `aggregate` | Querying | Quick rollups and aggregations |
| `execute_sql` | Querying | Raw SQL for complex queries |
| `insert_rows` | DML | Append new rows |
| `merge_rows` | DML | Upsert (insert + update + delete by PK) |
| `delete_rows` | DML | Delete rows by PK values |
| `import_csv` | Import | One-step CSV → Delta table |
| `infer_schema` | Import | Preview CSV schema before import |
| `set_column_domain` | Domains | Set fixed/hierarchical/reference domain |
| `get_allowed_values` | Domains | Get allowed values for a domain column |
| `delete_column_domain` | Domains | Remove domain constraint |
| `validate_domain_values` | Quality | Scan for domain violations |
| `validate_constraints` | Quality | Check all table constraints |
| `get_table_history` | Version Control | List all versions with commit metadata |
| `diff_table_versions` | Version Control | Compare two versions (inserts/deletes/updates) |
| `rollback_table` | Version Control | Revert to a previous version |
| `restore_table` | Version Control | Restore to a UTC timestamp |
| `get_audit_log` | Audit | Organisation-level activity log |
| `optimize_table` | Maintenance | Compact small files |
| `vacuum_table` | Maintenance | Clean up old files beyond retention |
| `register_s3_storage` | Storage config | Register an S3 (or S3-compatible) storage connection |
| `register_az_storage` | Storage config | Register an Azure Blob storage connection |
| `register_gcs_storage` | Storage config | Register a Google Cloud Storage connection |
| `list_storage_configs` | Storage config | List registered storage connections |
| `update_storage_config` | Storage config | Update a storage connection's settings |
| `delete_storage_config` | Storage config | Remove a storage connection by `config_id` |
| `list_objects` | Object store | List objects/prefixes under a path |
| `download_object` | Object store | Download an object's contents |
| `upload_object` | Object store | Upload an object to a path |
| `delete_object` | Object store | Delete an object |
| `create_folder` | Object store | Create a folder/prefix |
| `move_objects` | Object store | Move/rename objects within a config |
| `create_demo_table` | Object store | Seed a demo Delta table in a storage config |

## Universal gotchas (apply to most tools)

* `table` parameter = fully-qualified `catalog.schema.table`. Get from `list_tables`.
* `table_name` in `create_table`/`import_csv` = bare name only (`products`), never prefixed.
* `execute_sql` table names must be backtick-quoted per component: `` `catalog`.`schema`.`table` ``.
* `location` = full storage URL, not relative path.
* `destination_path` in rename/move = relative path, not URL.
