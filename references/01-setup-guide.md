# Setup Guide: Unity Catalog + Snowflake CLD

## Prerequisites

### Databricks Side
- Databricks account on AWS, Azure, or Google Cloud
- Workspace with Unity Catalog enabled
- Metastore admin or CREATE CATALOG privilege
- Account/workspace admin for service principal creation

### Snowflake Side
- Role with CREATE CATALOG INTEGRATION + CREATE DATABASE privileges
- ACCOUNTADMIN or equivalent for initial setup

## Step-by-Step Setup

### 1. Create a Service Principal in Databricks

Create a service principal for Snowflake authentication:
- Databricks → Settings → Identity and access → Service principals → Add
- Copy the **Application ID** — you'll need it for Snowflake

Create an **OAuth secret** for the service principal:
- Select the service principal → Secrets → Generate secret
- Copy the **Secret** value immediately (shown only once)

### 2. Grant Unity Catalog Privileges to the Service Principal

```sql
-- In Databricks
GRANT CREATE TABLE ON CATALOG <catalog_name> TO `<application_id>`;
GRANT EXTERNAL USE SCHEMA ON CATALOG <catalog_name> TO `<application_id>`;
GRANT MODIFY ON CATALOG <catalog_name> TO `<application_id>`;
GRANT SELECT ON CATALOG <catalog_name> TO `<application_id>`;
GRANT USE CATALOG ON CATALOG <catalog_name> TO `<application_id>`;
GRANT USE SCHEMA ON CATALOG <catalog_name> TO `<application_id>`;
```

### 3. Enable External Data Access on Metastore

In Databricks, the metastore admin must enable external data access. See Databricks docs for your cloud provider.

### 4. Create Catalog Integration in Snowflake (Vended Credentials)

```sql
CREATE CATALOG INTEGRATION unity_catalog_int
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  CATALOG_NAMESPACE = '<catalog_name>'
  REST_CONFIG = (
    CATALOG_URI = 'https://<workspace-url>/api/2.1/unity-catalog/iceberg'
    WAREHOUSE = '<catalog_name>'
  )
  REST_AUTHENTICATION = (
    TYPE = OAUTH
    OAUTH_TOKEN_URI = 'https://<workspace-url>/oidc/v1/token'
    OAUTH_CLIENT_ID = '<application_id>'
    OAUTH_CLIENT_SECRET = '<oauth_secret>'
    OAUTH_ALLOWED_SCOPES = ('all-apis')
  )
  ENABLED = TRUE;
```

### 5. Create the Catalog-Linked Database

```sql
CREATE DATABASE my_unity_cld
  LINKED_CATALOG = (
    CATALOG = 'unity_catalog_int'
  );
```

For writable access (bidirectional):
```sql
CREATE DATABASE my_unity_cld
  LINKED_CATALOG = (
    CATALOG = 'unity_catalog_int',
    ALLOWED_WRITE_OPERATIONS = ALL
  );
```

### 6. Verify the Setup

```sql
-- Check configuration
SELECT SYSTEM$GET_CATALOG_LINKED_DATABASE_CONFIG('my_unity_cld');

-- Check sync status
SELECT SYSTEM$CATALOG_LINK_STATUS('my_unity_cld');

-- List discovered tables
SHOW ICEBERG TABLES IN DATABASE my_unity_cld;
```

## Optional: Limit Discovery Scope

```sql
-- Only sync specific namespaces
CREATE DATABASE my_unity_cld
  LINKED_CATALOG = (
    CATALOG = 'unity_catalog_int',
    ALLOWED_NAMESPACES = ('schema1', 'schema2')
  );

-- Block specific namespaces
CREATE DATABASE my_unity_cld
  LINKED_CATALOG = (
    CATALOG = 'unity_catalog_int',
    BLOCKED_NAMESPACES = ('staging', 'temp')
  );
```

## Reference
- [Tutorial: Bidirectional access to Unity Catalog](https://docs.snowflake.com/en/user-guide/tutorials/tables-iceberg-set-up-bidirectional-access-to-unity-catalog)
- [Configure catalog integration for Unity Catalog](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-rest-unity)
