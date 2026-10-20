# Known Limitations: Unity Catalog CLDs

## Not Supported in CLDs

| Feature | Status | Notes |
|---|---|---|
| Cloning | Not supported | Cannot CLONE tables in a CLD |
| Replication | Not supported | Cannot replicate a CLD to another account |
| UNDROP ICEBERG TABLE | Not supported | Cannot recover dropped tables |
| Structured type ALTER COLUMN | Not supported | Cannot evolve structured types (OBJECT/ARRAY/MAP) in CLD tables |
| Time Travel (manual control) | Limited | Retention is controlled by the external catalog, not Snowflake's DATA_RETENTION_TIME_IN_DAYS |
| Altering table properties | Not supported | Remote catalog table properties (retention, buffers) aren't synced and can't be altered from Snowflake |
| Non-Iceberg objects | Not supported | Only schemas, Iceberg tables, and database roles can be created in a CLD |
| Sharing with listings | Not supported | Cannot share CLD tables via Snowflake Marketplace listings |
| Direct sharing | Supported | Can share CLD tables via direct shares |

## Auto-Refresh Limitations

- DDL-only schema changes (especially column drops) don't auto-sync — see `references/03-column-drop-issue.md`
- Toggling auto-refresh on/off frequently can slow metadata refreshes
- `AUTO_REFRESH` with `INFER_SCHEMA` is not supported
- v2 → v3 format version upgrades from external engines break both manual and automated refresh

## Write Limitations

- `DROP ICEBERG TABLE` in Snowflake with write enabled propagates to Unity Catalog — removes table and data from both systems
- Creating/writing in nested namespaces requires catalog support for nested namespaces + `NAMESPACE_MODE = FLATTEN_NESTED_NAMESPACE`
- Equality delete files not supported (only position deletes)
- Don't use a period (`.`) in namespace names unless it's the flatten delimiter

## Discovery Limitations

- Snowflake does NOT sync remote catalog access control (users/roles from Unity Catalog)
- Namespace flatten delimiter character cannot appear in remote namespace names (those namespaces are skipped)
- The `BLOCKED_NAMESPACES` list takes precedence over `ALLOWED_NAMESPACES`

## Identifier Limitations

- For AWS Glue catalogs, you MUST use lowercase + double quotes for all identifiers
- Two identifiers that differ only in casing cannot coexist when using `CASE_INSENSITIVE` mode

## Billing Considerations

- Cloud services usage (discovery) is charged only if daily consumption exceeds 10% of daily warehouse usage
- Auto-refresh costs appear as Snowpipe charges in PIPE_USAGE_HISTORY
- Longer SYNC_INTERVAL_SECONDS reduces discovery credit consumption
- Event table logging for auto-refresh monitoring incurs additional costs

## Iceberg Version Considerations

- Default Iceberg version is v2 (configurable to v3 via `ICEBERG_VERSION_DEFAULT`)
- Downgrading format versions is NOT supported per the Apache Iceberg specification
- Before upgrading to v3, ensure all readers/writers support v3

## Masking and Tags

- Masking policies: **Supported** on CLD tables
- Tags: **Supported** on CLD tables
- Row access policies: **Supported** on CLD tables
- Aggregation policies: **Supported** on CLD tables

## Reference
- [Use a catalog-linked database — Considerations](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database)
- [CREATE DATABASE (catalog-linked) — Usage notes](https://docs.snowflake.com/en/sql-reference/sql/create-database-catalog-linked)
- [Iceberg v3 considerations](https://docs.snowflake.com/en/user-guide/tables-iceberg-v3-specification-support)
