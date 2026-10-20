# Sync Behavior: Discovery vs Auto-Refresh

A catalog-linked database uses **two independent sync mechanisms**. Understanding which handles what is critical for troubleshooting.

## Mechanism 1: Automatic Table Discovery

**What it does:** Polls the remote catalog to detect namespace and table-level changes.

**Handles:**
- New tables appearing in Unity Catalog → created in Snowflake
- Tables dropped from Unity Catalog → dropped from Snowflake (async)
- Tables renamed in Unity Catalog → old table dropped, new table created
- Namespaces added/removed

**Does NOT handle:**
- Column-level schema changes
- Data changes within tables

**Configuration:**
```sql
-- Default poll interval: 30 seconds
-- Change interval (30 to 86400 seconds):
ALTER DATABASE my_cld UPDATE LINKED_CATALOG
  SET SYNC_INTERVAL_SECONDS = 60;

-- Suspend discovery (saves credits):
ALTER DATABASE my_cld SUSPEND DISCOVERY;

-- Resume discovery:
ALTER DATABASE my_cld RESUME DISCOVERY;
```

**Check status:**
```sql
SELECT SYSTEM$CATALOG_LINK_STATUS('my_cld');
-- executionState should be RUNNING
```

## Mechanism 2: Automated Refresh (Per-Table)

**What it does:** Polls for new Iceberg snapshots on individual tables to sync data and schema changes.

**Handles:**
- New data from DML (INSERT, UPDATE, DELETE)
- Schema changes **that accompany DML** (column add/drop/rename bundled with a data write)
- Table and column description propagation (bidirectional)

**Does NOT handle:**
- DDL-only schema changes that don't produce a new snapshot (notably: column drops)

**Key documented limitation:**
> "Automated refresh synchronizes schema changes alongside DML operations such as INSERT, UPDATE, or DELETE. To apply schema changes made through DDL operations alone, perform a manual refresh."
> — [Snowflake docs](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)

**Configuration:**
- Auto-refresh is enabled by default for CLD tables
- Refresh interval is controlled at the catalog integration level via `REFRESH_INTERVAL_SECONDS`

## What Auto-Syncs — Complete Reference

| Change in Unity Catalog | Mechanism | Auto-synced? | Action needed |
|---|---|---|---|
| Add table | Discovery | Yes | None |
| Drop table | Discovery | Yes (async) | None |
| Rename table | Discovery | Yes (drop + create) | None |
| Add namespace | Discovery | Yes | None |
| Drop namespace | Discovery | Yes | None |
| Add column (DDL) | Auto-refresh | Yes | None |
| Drop column (DDL-only) | Auto-refresh | **No** | `ALTER ICEBERG TABLE ... REFRESH` |
| DML (INSERT/UPDATE/DELETE) | Auto-refresh | Yes | None |
| Schema change + DML | Auto-refresh | Yes | None |
| Table/column descriptions | Auto-refresh | Yes (bidirectional) | None |
| table-uuid mismatch | Auto-refresh | Drops table from CLD | Investigate root cause |

## Namespace Filtering

Control which Unity Catalog namespaces are synced:

```sql
-- Add namespaces to allow list
ALTER DATABASE my_cld UPDATE LINKED_CATALOG
  ADD ('sales', 'marketing') TO ALLOWED_NAMESPACES;

-- Remove from allow list
ALTER DATABASE my_cld UPDATE LINKED_CATALOG
  REMOVE ('staging') FROM ALLOWED_NAMESPACES;

-- Block specific namespaces (takes precedence over allowed)
ALTER DATABASE my_cld UPDATE LINKED_CATALOG
  ADD ('temp', 'sandbox') TO BLOCKED_NAMESPACES;

-- Reset to sync all namespaces
ALTER DATABASE my_cld UPDATE LINKED_CATALOG
  UNSET ALLOWED_NAMESPACES;
```

## Description Sync Behavior

- Snowflake bidirectionally syncs table and column descriptions
- Sync can update a description to a new value
- Sync **never replaces a non-empty description with an empty one**
- Other remote catalog table properties (retention policies, buffers) are NOT synced

## Reference
- [Use a catalog-linked database](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database)
- [Automatically refresh Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)
- [ALTER DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/alter-database-catalog-linked)
