# Monitoring: CLD Health, Refresh, and Costs

## SYSTEM Functions

### CLD Link Status (Discovery Health)

```sql
SELECT SYSTEM$CATALOG_LINK_STATUS('my_cld');
```

Returns JSON with:
- `executionState`: RUNNING, SUSPENDED, FAILED
- Discovery sync details

### CLD Configuration

```sql
SELECT SYSTEM$GET_CATALOG_LINKED_DATABASE_CONFIG('my_cld');
```

Returns: catalog integration name, catalog namespace, sync interval, write permissions.

### Auto-Refresh Status (Per Table)

```sql
SELECT SYSTEM$AUTO_REFRESH_STATUS('my_cld.my_schema.my_table');
```

Returns: execution state, pending snapshot count, last refresh time.

### Bulk Check via SHOW ICEBERG TABLES

```sql
SHOW ICEBERG TABLES IN DATABASE my_cld;
-- auto_refresh_status column shows status for each table
```

## Refresh History

```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY(
  TABLE_NAME => 'my_table'
));
```

Shows snapshot metadata and refresh timestamps.

## Event Table Monitoring

Set up event logging for auto-refresh events:

```sql
-- Enable DEBUG-level events for a table
ALTER ICEBERG TABLE my_table SET LOG_EVENT_LEVEL = DEBUG;

-- Query auto-refresh events
SELECT
  record_type,
  record:"name" AS event_name,
  record:"severity_text" AS log_level,
  resource_attributes:"snow.database.name" AS database_name,
  resource_attributes:"snow.schema.name" AS schema_name,
  resource_attributes:"snow.table.name" AS table_name,
  record_attributes:"snow.snapshot.id" AS snapshot_id,
  parse_json(value):snapshot_state AS snapshot_state
FROM my_event_table
WHERE record_type = 'EVENT'
  AND event_name = 'iceberg_auto_refresh_snapshot_lifecycle';
```

### Detect Stale Tables

```sql
SET STALENESS_THRESHOLD_MINUTES = 20;

WITH last_successful_refresh AS (
  SELECT
    resource_attributes:"snow.table.name"::STRING AS table_name,
    MAX(timestamp) AS last_success_timestamp
  FROM my_event_table
  WHERE record:"name" = 'iceberg_auto_refresh_snapshot_lifecycle'
    AND parse_json(value):snapshot_state::STRING = 'completed'
  GROUP BY table_name
)
SELECT table_name, last_success_timestamp
FROM last_successful_refresh
WHERE last_success_timestamp < DATEADD(minute, -$STALENESS_THRESHOLD_MINUTES, CURRENT_TIMESTAMP())
ORDER BY last_success_timestamp ASC;
```

## Credit Usage Tracking

```sql
-- CLD-specific credit usage (last 12 months)
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.CATALOG_LINKED_DATABASE_USAGE_HISTORY
WHERE START_TIME >= DATEADD(day, -30, CURRENT_TIMESTAMP())
ORDER BY START_TIME DESC;
```

Billing breakdown:
- **Discovery** (table/schema add/drop): CREDITS_USED_CLOUD_SERVICES
- **Table creation**: CREDITS_USED_COMPUTE (auto-refresh pipeline)
- **Auto-refresh**: Same as Snowpipe charges (PIPE_USAGE_HISTORY)

```sql
-- Auto-refresh costs (appears as Snowpipe)
SELECT pipe_name, credits_used, start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
ORDER BY start_time DESC;
```

## Snowsight Dashboard

Navigate to: **Horizon Catalog → Catalog → Connections → Data connections → Catalog-linked databases**

The dashboard shows:
- **Link status**: Running/Failed for each CLD
- **Table creation issues**: Count of tables that failed to create
- **Table refresh issues**: Tables with STOPPED/STALLED/Not initialized status
- **Last discovery** and **Last refresh** timestamps

## Reference
- [Monitor CLDs in Snowsight](https://docs.snowflake.com/en/user-guide/tables-iceberg-monitor-catalog-linked-databases)
- [SYSTEM$CATALOG_LINK_STATUS](https://docs.snowflake.com/en/sql-reference/functions/system_catalog_link_status)
- [SYSTEM$AUTO_REFRESH_STATUS](https://docs.snowflake.com/en/sql-reference/functions/system_auto_refresh_status)
- [CATALOG_LINKED_DATABASE_USAGE_HISTORY](https://docs.snowflake.com/en/sql-reference/account-usage/catalog_linked_database_usage_history)
