# Unity Catalog + Snowflake: Catalog-Linked Database Guide

A comprehensive knowledge base for using **Databricks Unity Catalog** with **Snowflake** via **Catalog-Linked Databases (CLDs)**.

Covers setup, sync behavior, schema evolution, bidirectional writes, troubleshooting, monitoring, identifier resolution, and known limitations — all validated against official Snowflake documentation.

## Structure

```
├── SKILL.md                              # CoCo skill router (intent detection → reference files)
├── references/
│   ├── 01-setup-guide.md                 # End-to-end setup (service principal, catalog integration, CLD)
│   ├── 02-sync-behavior.md               # Discovery vs auto-refresh — what syncs and what doesn't
│   ├── 03-column-drop-issue.md           # Column drops not syncing (root cause + workarounds)
│   ├── 04-schema-evolution.md            # Full schema evolution matrix (add/drop/rename/type widen)
│   ├── 05-write-support.md               # Bidirectional writes — DML from Snowflake → Unity Catalog
│   ├── 06-troubleshooting.md             # STOPPED/STALLED recovery, v2→v3, table-uuid mismatch
│   ├── 07-monitoring.md                  # SYSTEM$ functions, event tables, credit tracking, Snowsight
│   ├── 08-identifier-resolution.md       # Case sensitivity, quoting rules, DDL normalization
│   └── 09-limitations.md                 # Known limitations, unsupported features, gotchas
```

## Quick Reference

```sql
-- Check CLD link status
SELECT SYSTEM$CATALOG_LINK_STATUS('<db_name>');

-- Check CLD configuration
SELECT SYSTEM$GET_CATALOG_LINKED_DATABASE_CONFIG('<db_name>');

-- Check auto-refresh status for a table
SELECT SYSTEM$AUTO_REFRESH_STATUS('<db>.<schema>.<table>');

-- Manual refresh (fixes DDL-only schema changes like column drops)
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;

-- Force re-discovery
ALTER DATABASE <db_name> RESUME DISCOVERY;
```

## Key Findings

### Column Drops Don't Auto-Sync

When you drop a column in Unity Catalog, it does **not** automatically reflect in Snowflake. Adding columns works fine.

**Root cause:** Snowflake's auto-refresh syncs schema changes only alongside DML operations. Column drops are DDL-only and may not produce a new Iceberg snapshot.

**Fix:**
```sql
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;
```

See [references/03-column-drop-issue.md](references/03-column-drop-issue.md) for the full analysis.

### Two Sync Mechanisms

| Mechanism | Handles | Does NOT Handle |
|---|---|---|
| **Automatic Table Discovery** | Tables, namespaces (add/drop/rename) | Column-level changes |
| **Automated Refresh** | Data snapshots + schema changes with DML | DDL-only schema changes (column drops) |

See [references/02-sync-behavior.md](references/02-sync-behavior.md) for the complete sync behavior matrix.

## Snowflake Documentation References

- [Use a catalog-linked database](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database)
- [Tutorial: Bidirectional access to Unity Catalog](https://docs.snowflake.com/en/user-guide/tutorials/tables-iceberg-set-up-bidirectional-access-to-unity-catalog)
- [Automatically refresh Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)
- [Write support for externally managed Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-externally-managed-writes)
- [ALTER DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/alter-database-catalog-linked)
- [Monitor CLDs in Snowsight](https://docs.snowflake.com/en/user-guide/tables-iceberg-monitor-catalog-linked-databases)

## Using as a CoCo Skill

This repo doubles as a [Snowflake CoCo](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill. The `SKILL.md` file acts as a router that detects intent and loads the appropriate reference file.

To install:
1. Copy the entire folder to `~/.snowflake/cortex/skills/agent-created/snowflake/unity-catalog-snowflake/`
2. Invoke with `$unity-catalog-snowflake` in CoCo

## Contributing

Found an inaccuracy or want to add coverage for a new topic? PRs welcome. All content should be validated against the [official Snowflake documentation](https://docs.snowflake.com).

## License

MIT
