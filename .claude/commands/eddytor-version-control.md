Use this skill when helping the user view table history, undo changes, rollback to a previous version, restore to a point in time, diff versions, or check the audit log in Eddytor.

# Version Control & Time Travel

Every mutation creates a new Delta Lake version. You can inspect history, rollback to any version, or restore to any timestamp.

## Procedure: investigating and reverting a problem

1. `get_table_history` → review recent versions (operation, timestamp, user, metrics, comment)
2. Identify the last good version number or timestamp
3. Confirm with user before proceeding — rollback/restore are **destructive**
4. `rollback_table(table, version=N)` or `restore_table(table, timestamp="2024-06-15T14:30:00Z")`
5. `profile_table` + `validate_constraints` → confirm restored state

**Use** `rollback_table` when the user knows the exact version.
**Use** `restore_table` when the user knows the approximate time.

## Diff between versions

`diff_table_versions` compares two versions row-by-row — like `git diff` for table data.
* Requires at least one primary key column
* Returns `changeType` per row (`INSERT`, `DELETE`, `UPDATE`), `current`/`previous` values, and `changedColumns` for updates
* Handles schema evolution by comparing only columns present in both versions

Workflow: `get_table_history` → `diff_table_versions` → decide whether to `rollback_table`

## Audit trail

`get_audit_log` returns org-level entries. Filter by `table`, `action`, `from`/`to` timestamps.

Leave breadcrumbs with `merge_rows`:
```json
{ "comment": "Q1 price adjustment — approved by finance team", ... }
```

Comments appear in `get_table_history` and make finding rollback points much easier.

## Gotchas

* `rollback_table` and `restore_table` are **destructive** — all versions after the target are discarded. Always check history first.
* After `vacuum_table` with low retention, time-travel to old versions **fails** because underlying files are deleted. Default retention is 168 hours (7 days).
* `optimize_table` creates a version in history but doesn't change data — don't mistake it for a data change when reviewing history.
* `restore_table` timestamp must be ISO 8601 UTC (e.g., `"2024-06-15T14:30:00Z"`).

## Plan-validate-execute pattern for destructive operations

Before bulk deletes, schema changes, or domain changes:
1. `get_table_history` → note current version as your safety net
2. Execute the operation
3. `validate_constraints` + `profile_table` → confirm result
4. If something went wrong → `rollback_table` to the noted version

## Guidelines

* Always `get_table_history` before rollback/restore — never guess version numbers
* Confirm with user before executing — destructive operations have no undo
* Use `merge_rows` with `comment` for all significant changes
* After restore, validate: `profile_table` + `validate_constraints`
