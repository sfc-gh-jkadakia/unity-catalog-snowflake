# Column Drop Issue: DDL-Only Drops Not Syncing

## Problem

When a column is dropped in Unity Catalog, the change does not automatically reflect in the Snowflake catalog-linked database. The dropped column continues to appear. Adding new columns works fine — the issue is specific to column drops.

## Root Cause

Snowflake's automated refresh syncs schema changes **only alongside DML operations** (INSERT, UPDATE, DELETE). Dropping a column is DDL-only and may not produce a new Iceberg snapshot, so auto-refresh has nothing to detect.

### Why Add Column Works But Drop Column Does Not

Both are DDL operations, but they behave asymmetrically:

- **Add column** is a forward-compatible (additive) change in Iceberg. Snowflake can safely extend its schema when it encounters new field IDs in metadata. Existing data files remain valid; the new column returns NULL for older data.
- **Drop column** is non-additive. The column's field ID still exists in historical schemas and older data files. Snowflake must actively reconcile and remove the column, which requires reading the updated `current-schema-id`. Without a new snapshot, auto-refresh does not detect the change.

### Snowflake Doc Reference

> "Automated refresh synchronizes schema changes alongside DML operations such as INSERT, UPDATE, or DELETE. To apply schema changes made through DDL operations alone, perform a manual refresh."
> — [Automatically refresh Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)

**Note:** The asymmetry between add and drop column is observed behavior consistent with Iceberg spec semantics. Snowflake docs do not explicitly document this asymmetry — they state broadly that DDL-only changes require manual refresh.

## Workarounds

### Option 1 — Manual Refresh (Preferred)

```sql
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;
```

Forces Snowflake to re-read the latest metadata from Unity Catalog, including the column drop.

### Option 2 — Trigger a DML in Databricks

After dropping the column, run any write operation in Databricks:

```sql
-- In Databricks: trivial update to create a new snapshot
UPDATE <catalog>.<schema>.<table> SET <some_col> = <some_col> WHERE 1=0;
-- Or: insert and delete a dummy row
```

This creates a new Iceberg snapshot that auto-refresh picks up on the next poll (default 30s).

### Option 3 — Recreate the Table

As a last resort:

```sql
CREATE OR REPLACE ICEBERG TABLE <db>.<schema>.<table>
  CATALOG_TABLE_NAME = '<remote_table_name>'
  CATALOG = '<catalog_integration_name>'
  AUTO_REFRESH = TRUE;
```

## Diagnostic Steps

```sql
-- 1. Check auto-refresh status
SELECT SYSTEM$AUTO_REFRESH_STATUS('<db>.<schema>.<table>');

-- 2. Check CLD link status
SELECT SYSTEM$CATALOG_LINK_STATUS('<db_name>');

-- 3. Verify current columns in Snowflake
DESCRIBE ICEBERG TABLE <db>.<schema>.<table>;

-- 4. Check refresh history
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY(
  TABLE_NAME => '<table_name>'
));
```
