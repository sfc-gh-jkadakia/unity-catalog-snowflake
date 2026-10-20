---
name: unity-catalog-snowflake
description: "Comprehensive guide for using Databricks Unity Catalog with Snowflake via catalog-linked databases (CLD). Covers setup, sync behavior, schema evolution, write support, troubleshooting, monitoring, identifier resolution, and known limitations. Triggers: Unity Catalog, Databricks Iceberg, CLD setup, catalog linked database, Unity Catalog column drop, CLD sync, CLD troubleshoot, Unity Catalog Snowflake, Databricks Snowflake integration, CLD schema evolution, CLD refresh, CLD write support, bidirectional Iceberg, Databricks to Snowflake, column not syncing CLD, CLD monitoring, CLD limitations, CATALOG_CASE_SENSITIVITY, vended credentials Unity Catalog."
---

# Unity Catalog + Snowflake (Catalog-Linked Databases)

## Intent Detection

Route to the appropriate reference file based on the user's question:

| Intent | Triggers | Reference File |
|--------|----------|----------------|
| **Setup** | setup, configure, connect, create CLD, service principal, catalog integration, getting started | `references/01-setup-guide.md` |
| **Sync behavior** | how does sync work, discovery, auto-refresh, what syncs, sync interval, ALLOWED_NAMESPACES | `references/02-sync-behavior.md` |
| **Column drop issue** | column drop not syncing, dropped column still appears, column missing | `references/03-column-drop-issue.md` |
| **Schema evolution** | schema change, add column, rename column, type widening, partition evolution | `references/04-schema-evolution.md` |
| **Write support** | write to Unity Catalog, INSERT from Snowflake, bidirectional, DML, merge-on-read | `references/05-write-support.md` |
| **Troubleshooting** | error, failed, STOPPED, STALLED, refresh failing, v2 to v3, table-uuid mismatch | `references/06-troubleshooting.md` |
| **Monitoring** | monitor, status, event table, refresh history, credit usage, Snowsight dashboard | `references/07-monitoring.md` |
| **Identifiers** | case sensitivity, quoting, double quotes, CASE_INSENSITIVE, identifier resolution | `references/08-identifier-resolution.md` |
| **Limitations** | limitation, not supported, unsupported, restriction, gotcha, caveat | `references/09-limitations.md` |

## Workflow

1. Detect user intent from the table above.
2. Load the matching reference file using `skill_view`.
3. If multiple intents, load multiple files.
4. If unclear, ask the user which area they need help with.

## Quick Reference — Key SQL Commands

```sql
-- Check CLD link status
SELECT SYSTEM$CATALOG_LINK_STATUS('<db_name>');

-- Check CLD configuration
SELECT SYSTEM$GET_CATALOG_LINKED_DATABASE_CONFIG('<db_name>');

-- Check auto-refresh status for a table
SELECT SYSTEM$AUTO_REFRESH_STATUS('<db>.<schema>.<table>');

-- Manual refresh (fixes DDL-only schema changes)
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;

-- Force re-discovery
ALTER DATABASE <db_name> RESUME DISCOVERY;
```

## Key Documentation Links

- [Use a catalog-linked database](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database)
- [Tutorial: Bidirectional access to Unity Catalog](https://docs.snowflake.com/en/user-guide/tutorials/tables-iceberg-set-up-bidirectional-access-to-unity-catalog)
- [Automatically refresh Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)
- [Write support for externally managed Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-externally-managed-writes)
- [ALTER DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/alter-database-catalog-linked)
- [Monitor CLDs in Snowsight](https://docs.snowflake.com/en/user-guide/tables-iceberg-monitor-catalog-linked-databases)
