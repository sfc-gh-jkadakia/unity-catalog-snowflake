# Troubleshooting: Unity Catalog CLD Issues

## Auto-Refresh States

| State | Meaning | Action |
|---|---|---|
| `RUNNING` | Healthy, refresh is working | None |
| `STALLED` | Snowflake is attempting recovery | Wait; may auto-recover to RUNNING |
| `STOPPED` | Unrecoverable error; refresh halted | Manual intervention required |
| `ICEBERG_TABLE_NOT_INITIALIZED` | Table synced but initial refresh failed | Fix underlying issue, re-enable auto-refresh |

## STOPPED Auto-Refresh Recovery

```sql
-- 1. Turn off auto-refresh
ALTER ICEBERG TABLE <db>.<schema>.<table> SET AUTO_REFRESH = FALSE;

-- 2. Perform manual refresh
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;

-- 3. Re-enable auto-refresh
ALTER ICEBERG TABLE <db>.<schema>.<table> SET AUTO_REFRESH = TRUE;

-- 4. Verify status
SELECT SYSTEM$AUTO_REFRESH_STATUS('<db>.<schema>.<table>');
-- pendingSnapshotCount should gradually decrease
```

## Iceberg v2 → v3 Upgrade Failure

If another engine (e.g., Databricks) upgrades a table from Iceberg v2 to v3, both manual and automated refresh will fail.

**Fix:**
1. In Databricks, perform a DML operation to create a new snapshot (just changing table properties is not enough)
2. In Snowflake, recreate the table:
```sql
CREATE OR REPLACE ICEBERG TABLE <db>.<schema>.<table>
  CATALOG_TABLE_NAME = '<remote_table_name>'
  CATALOG = '<catalog_integration_name>'
  AUTO_REFRESH = TRUE;
```

## table-uuid Mismatch

If the `table-uuid` of the external table and the CLD table don't match, refresh fails and Snowflake **drops the table from the CLD** (does not affect the remote table).

This can happen if the table was recreated in Unity Catalog with a different UUID.

**Fix:** The table will be re-discovered on the next discovery cycle if it still exists in the remote catalog.

## Table Created But Not Initialized

Tables that sync successfully but fail to refresh automatically have `ICEBERG_TABLE_NOT_INITIALIZED` status.

```sql
-- Find these tables
SHOW ICEBERG TABLES IN DATABASE my_cld;
-- Check auto_refresh_status column
```

**Causes:** Corrupted data files, permission issues, unsupported data types.

**Fix:**
1. Resolve the error in the remote catalog (fix corrupt files, grant permissions)
2. Re-enable auto-refresh:
```sql
ALTER ICEBERG TABLE <table> SET AUTO_REFRESH = TRUE;
```

## Discovery Failures

```sql
-- Check discovery status
SELECT SYSTEM$CATALOG_LINK_STATUS('my_cld');
```

Common issues:
- **SUSPENDED**: Discovery was manually suspended → `ALTER DATABASE my_cld RESUME DISCOVERY;`
- **FAILED**: Catalog integration misconfigured, permissions revoked, or network issues
- **Tables missing**: Check ALLOWED_NAMESPACES / BLOCKED_NAMESPACES filters

## DDL-Only Schema Changes Not Syncing

See `references/03-column-drop-issue.md` for the detailed analysis. Quick fix:

```sql
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;
```

## Permission Errors After BCR-2114

After Snowflake's BCR-2114 behavior change, the role with OWNERSHIP on the CLD must also have USAGE on both the CATALOG INTEGRATION and EXTERNAL VOLUME (if used).

```sql
GRANT USAGE ON INTEGRATION unity_catalog_int TO ROLE cld_owner_role;
GRANT USAGE ON EXTERNAL VOLUME my_ext_vol TO ROLE cld_owner_role;  -- if applicable
```

## Reference
- [Automatically refresh Iceberg tables — Error recovery](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)
- [Iceberg v3 considerations](https://docs.snowflake.com/en/user-guide/tables-iceberg-v3-specification-support)
- [BCR-2114: USAGE privilege required](https://docs.snowflake.com/en/release-notes/bcr-bundles/2025_07/bcr-2114)
