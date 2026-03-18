Use this skill when helping the user improve table read performance, compact files, clean up storage, or diagnose slow queries in Eddytor.

# Table Optimization & Maintenance

## Default procedure: optimize → vacuum → verify

1. `profile_table` → understand current state (row count, data shape)
2. `optimize_table` → compact small files into larger ones
3. `vacuum_table(dry_run=true)` → preview what would be deleted
4. Review file count and space savings
5. `vacuum_table(dry_run=false)` → execute cleanup
6. `get_table_history` → confirm both operations completed

**Order matters:** optimize first (creates compacted files), then vacuum (removes old small files).

## `optimize_table`

Merges small Parquet files. Run after many small writes (dozens of inserts/merges).

```json
{ "table": "eddytor.cfg_xxx.abc123_products", "target_size_mb": 256 }
```

Idempotent — running twice on an optimized table is harmless. Creates a new version but doesn't change data.

## `vacuum_table`

Deletes old unreferenced files. **Always dry-run first.**

```json
{ "table": "eddytor.cfg_xxx.abc123_products", "retention_hours": 168, "dry_run": true }
```

## Gotchas

* Setting `retention_hours` below 168 may **break time-travel**. After vacuum, `rollback_table` and `restore_table` to versions older than retention will fail — the underlying files are gone.
* Vacuum is **NOT idempotent** — once files are deleted, they're gone forever. The dry run is your safety net.
* Don't optimize after every single write — batch writes first, then optimize periodically.
* If a table is small (<1000 rows), optimization has minimal benefit.
* `optimize_table` creates a version in history — don't mistake it for a data change.

## Diagnosing slow queries

Check in this order:
1. Are you selecting only needed columns? `query_rows` with `columns: [...]`
2. Are you filtering early? Use the `filter` parameter
3. Has the table had many small writes? → `optimize_table`
4. Is it a very large table? → `profile_table` to check row count, use more specific filters

## Guidelines

* Run `optimize_table` after sequences of many small writes — biggest single performance win
* Always dry-run `vacuum_table` first
* Keep `retention_hours >= 168` unless you're certain time-travel isn't needed
* Check `get_table_history` to see when last OPTIMIZE ran — avoid redundant runs
