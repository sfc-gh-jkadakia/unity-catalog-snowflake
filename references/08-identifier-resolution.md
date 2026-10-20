# Identifier Resolution: Case Sensitivity in CLDs

## Default for Unity Catalog: CASE_INSENSITIVE

Unity Catalog uses case-insensitive identifiers. When you create a CLD linked to Unity Catalog, the default is `CATALOG_CASE_SENSITIVITY = CASE_INSENSITIVE`.

This means:
- You do NOT need to double-quote identifiers in queries or DDL
- All of these are equivalent: `table1`, `TABLE1`, `Table1`, `"table1"`
- DDL from Snowflake normalizes names to **lowercase** on the remote catalog

## Querying Tables

```sql
-- All of these work for a table named "my_table" in Unity Catalog
SELECT * FROM my_table;
SELECT * FROM MY_TABLE;
SELECT * FROM My_Table;
SELECT * FROM "my_table";
```

## DDL Name Normalization

When creating objects from Snowflake, names are lowercased in Unity Catalog:

```sql
CREATE SCHEMA My_Schema;
-- Creates "my_schema" in Unity Catalog

CREATE ICEBERG TABLE My_Schema.My_Table (Col1 INT, Col2 STRING);
-- Creates "my_table" with columns "col1", "col2" in Unity Catalog

ALTER ICEBERG TABLE my_table ADD COLUMN New_Col STRING;
-- Adds "new_col" in Unity Catalog

ALTER ICEBERG TABLE my_table RENAME COLUMN col2 TO Aux_Column;
-- Renames to "aux_column" in Unity Catalog
```

## Quoted Identifiers and Mixed Case

When `QUOTED_IDENTIFIERS_IGNORE_CASE` is **FALSE** (the default):
- Double-quoted identifiers preserve exact casing
- `CREATE SCHEMA "MySchema"` creates `MySchema` (mixed case) in the remote catalog
- You must use the same quoted form to reference it: `"MySchema"`

When `QUOTED_IDENTIFIERS_IGNORE_CASE` is **TRUE**:
- Snowflake resolves all identifiers case-insensitively, whether quoted or not
- DDL still normalizes to lowercase on the remote catalog

## CASE_SENSITIVE Mode

If you set `CATALOG_CASE_SENSITIVITY = CASE_SENSITIVE` (e.g., for Snowflake Open Catalog):
- Snowflake matches identifiers exactly, including case
- Unquoted identifiers are converted to UPPERCASE by Snowflake
- You must double-quote identifiers to match the exact case in the remote catalog

```sql
-- If remote catalog has "Table1":
SELECT * FROM "Table1";       -- Valid
SELECT * FROM table1;          -- Invalid (resolves to TABLE1)
SELECT * FROM TABLE1;          -- Invalid (unless TABLE1 exists)
```

## Practical Advice for Unity Catalog

Since Unity Catalog is case-insensitive by default:
1. Don't quote identifiers — let Snowflake handle normalization
2. Use any casing in queries — they all resolve the same way
3. When creating objects from Snowflake, expect lowercase names in Unity Catalog
4. If you need mixed-case names, use double quotes (but then you must always quote them)

## Reference
- [Requirements for identifier resolution in a CLD](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database)
- [Apr 2026: Double-quoted identifiers no longer required for DDL](https://docs.snowflake.com/en/release-notes/2026/other/2026-04-30-cld-case-insensitive-ddl)
