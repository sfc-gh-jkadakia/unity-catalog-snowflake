# Schema Evolution: Unity Catalog ↔ Snowflake CLD

## Supported Schema Evolution (Iceberg Spec)

Iceberg supports the following schema evolution operations. Support in Snowflake depends on whether the table is Snowflake-managed or externally managed (CLD/Unity Catalog).

### From Unity Catalog Side (External Changes)

| Operation | Auto-syncs to Snowflake? | Notes |
|---|---|---|
| Add column | Yes | Forward-compatible; auto-refresh picks it up |
| Drop column | **No** (DDL-only) | Requires manual `ALTER ICEBERG TABLE ... REFRESH` |
| Rename column | Depends | Needs manual refresh if DDL-only |
| Type widening (int→long, float→double, decimal precision increase) | Depends | Needs manual refresh if DDL-only |
| Reorder columns | Depends | Needs manual refresh if DDL-only |
| Add/drop/replace partition | N/A | Partition evolution from external catalog — next DML-bearing snapshot syncs it |

**Rule of thumb:** If the schema change is accompanied by a DML operation that creates a new Iceberg snapshot, it syncs automatically. If it's DDL-only, run `ALTER ICEBERG TABLE ... REFRESH`.

### From Snowflake Side (CLD → Unity Catalog)

For **writable** CLDs (`ALLOWED_WRITE_OPERATIONS = ALL`):

| Operation | Supported? | Notes |
|---|---|---|
| `ALTER ICEBERG TABLE ... ADD COLUMN` | Yes | Commits to Unity Catalog |
| `ALTER ICEBERG TABLE ... DROP COLUMN` | Yes | Commits to Unity Catalog |
| `ALTER ICEBERG TABLE ... RENAME COLUMN` | Yes | Commits to Unity Catalog |
| `ALTER ICEBERG TABLE ... ALTER COLUMN SET DATA TYPE` (structured types) | **No** | Not supported for tables in a CLD |
| INSERT / UPDATE / DELETE / MERGE | Yes | Commits to Unity Catalog |

### Iceberg Type Widening Rules

Per the Apache Iceberg specification:
- `int` → `long`
- `float` → `double`
- `decimal(P, S)` → `decimal(P', S)` where P' > P

These are the only supported type promotions. Other type changes (e.g., string→int) are not supported.

## Case Sensitivity and Schema Evolution

When your CLD uses `CATALOG_CASE_SENSITIVITY = CASE_INSENSITIVE` (the default for Unity Catalog):

- DDL from Snowflake normalizes names to **lowercase** on the remote catalog
- `ALTER ICEBERG TABLE my_table ADD COLUMN New_Col STRING` → adds `new_col` in Unity Catalog
- `ALTER ICEBERG TABLE my_table RENAME COLUMN col2 TO Aux_Column` → renames to `aux_column`

## Manual Refresh for DDL-Only Changes

When schema changes are made in Unity Catalog without accompanying DML:

```sql
-- Refresh a single table
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;

-- Verify the schema after refresh
DESCRIBE ICEBERG TABLE <db>.<schema>.<table>;
```

## Reference
- [ALTER ICEBERG TABLE](https://docs.snowflake.com/en/sql-reference/sql/alter-iceberg-table)
- [ALTER ICEBERG TABLE ... REFRESH](https://docs.snowflake.com/en/sql-reference/sql/alter-iceberg-table-refresh)
- [Automatically refresh Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)
