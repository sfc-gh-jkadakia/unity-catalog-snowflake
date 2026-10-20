# Write Support: Snowflake → Unity Catalog

## Overview

Snowflake supports bidirectional writes with Unity Catalog through writable catalog-linked databases. You can INSERT, UPDATE, DELETE, and MERGE data from Snowflake, and the changes are committed to Unity Catalog.

## Prerequisites

1. Catalog integration with **vended credentials** (or external volume with ALLOW_WRITES = TRUE)
2. CLD created with `ALLOWED_WRITE_OPERATIONS = ALL`
3. Service principal in Databricks has MODIFY + CREATE TABLE privileges on the catalog

## Enable Write Access

```sql
-- At CLD creation
CREATE DATABASE my_unity_cld
  LINKED_CATALOG = (
    CATALOG = 'unity_catalog_int',
    ALLOWED_WRITE_OPERATIONS = ALL
  );

-- Or alter existing CLD
ALTER DATABASE my_unity_cld UPDATE LINKED_CATALOG
  SET ALLOWED_WRITE_OPERATIONS = ALL;
```

**Warning:** With write permissions enabled, `DROP ICEBERG TABLE` in Snowflake propagates to the remote catalog, removing the table and data from both systems.

## Supported Write Operations

### DML on Existing Tables

```sql
-- Insert data
INSERT INTO my_cld.my_schema.my_table VALUES (1, 'data', CURRENT_DATE());

-- Update data
UPDATE my_cld.my_schema.my_table SET col1 = 'new_value' WHERE id = 1;

-- Delete data
DELETE FROM my_cld.my_schema.my_table WHERE id = 1;

-- Merge
MERGE INTO my_cld.my_schema.target t
  USING my_cld.my_schema.source s ON t.id = s.id
  WHEN MATCHED THEN UPDATE SET t.col1 = s.col1
  WHEN NOT MATCHED THEN INSERT (id, col1) VALUES (s.id, s.col1);
```

### Create Tables from Snowflake

```sql
USE DATABASE my_cld;
USE SCHEMA my_namespace;

CREATE ICEBERG TABLE new_table (
  id INT,
  name STRING,
  created_at TIMESTAMP
)
TARGET_FILE_SIZE = '64MB';
```

This creates the table in both Snowflake and Unity Catalog.

### Create Namespaces from Snowflake

```sql
CREATE SCHEMA my_new_namespace;
-- Creates namespace in Unity Catalog + schema in Snowflake

DROP SCHEMA my_namespace;
-- Drops from both Unity Catalog and Snowflake
```

## Row-Level Deletes (Merge-on-Read)

Position row-level deletes are supported on S3, Azure, and GCS.

```sql
-- Control merge-on-read at database level
ALTER DATABASE my_cld SET ICEBERG_MERGE_ON_READ_BEHAVIOR = 'AUTO';

-- Or at table level
ALTER ICEBERG TABLE my_table SET ICEBERG_MERGE_ON_READ_BEHAVIOR = 'ENABLED';

-- Disable (forces copy-on-write mode)
ALTER ICEBERG TABLE my_table SET ICEBERG_MERGE_ON_READ_BEHAVIOR = 'DISABLED';
```

Values:
- `AUTO` — Snowflake decides based on workload
- `ENABLED` — Always use merge-on-read (position deletes)
- `DISABLED` — Always use copy-on-write

## Nested Namespaces

Creating tables in nested namespaces requires:
1. Catalog must support nested namespaces (Unity Catalog does)
2. `NAMESPACE_MODE = FLATTEN_NESTED_NAMESPACE` on the CLD
3. `NAMESPACE_FLATTEN_DELIMITER` set (e.g., `-` or `/`)
4. Parent namespace must already exist

```sql
-- Create nested namespace
CREATE SCHEMA "parent-child";
```

## Read-Only Mode

To make a CLD read-only (prevent accidental writes):

```sql
ALTER DATABASE my_cld UPDATE LINKED_CATALOG
  SET ALLOWED_WRITE_OPERATIONS = NONE;
```

## Reference
- [Write support for externally managed Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-externally-managed-writes)
- [CREATE DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/create-database-catalog-linked)
