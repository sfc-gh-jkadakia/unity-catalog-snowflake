# Unity Catalog + Snowflake: Catalog-Linked Database Guide

A comprehensive knowledge base for using **Databricks Unity Catalog** with **Snowflake** via **Catalog-Linked Databases (CLDs)**.

Covers setup, sync behavior, schema evolution, bidirectional writes, troubleshooting, monitoring, identifier resolution, and known limitations — all validated against official Snowflake documentation.

## Structure

```
├── SKILL.md                              # CoCo skill router (intent detection → reference files)
├── references/
│   ├── 01-setup-guide.md                 # End-to-end setup (service principal, catalog integration, CLD)
│   ├── 02-sync-behavior.md               # Discovery vs auto-refresh — what syncs and what doesn't
│   ├── 03-column-drop-issue.md           # Column drops not syncing (root cause + workarounds)
│   ├── 04-schema-evolution.md            # Full schema evolution matrix (add/drop/rename/type widen)
│   ├── 05-write-support.md               # Bidirectional writes — DML from Snowflake → Unity Catalog
│   ├── 06-troubleshooting.md             # STOPPED/STALLED recovery, v2→v3, table-uuid mismatch
│   ├── 07-monitoring.md                  # SYSTEM$ functions, event tables, credit tracking, Snowsight
│   ├── 08-identifier-resolution.md       # Case sensitivity, quoting rules, DDL normalization
│   └── 09-limitations.md                 # Known limitations, unsupported features, gotchas
```

## Quick Reference

```sql
-- Check CLD link status
SELECT SYSTEM$CATALOG_LINK_STATUS('<db_name>');

-- Check CLD configuration
SELECT SYSTEM$GET_CATALOG_LINKED_DATABASE_CONFIG('<db_name>');

-- Check auto-refresh status for a table
SELECT SYSTEM$AUTO_REFRESH_STATUS('<db>.<schema>.<table>');

-- Manual refresh (fixes DDL-only schema changes like column drops)
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;

-- Force re-discovery
ALTER DATABASE <db_name> RESUME DISCOVERY;
```

## Key Findings

### Column Drops Don't Auto-Sync

When you drop a column in Unity Catalog, it does **not** automatically reflect in Snowflake. Adding columns works fine.

**Root cause:** Snowflake's auto-refresh syncs schema changes only alongside DML operations. Column drops are DDL-only and may not produce a new Iceberg snapshot.

**Fix:**
```sql
ALTER ICEBERG TABLE <db>.<schema>.<table> REFRESH;
```

See [references/03-column-drop-issue.md](references/03-column-drop-issue.md) for the full analysis.

### Two Sync Mechanisms

| Mechanism | Handles | Does NOT Handle |
|---|---|---|
| **Automatic Table Discovery** | Tables, namespaces (add/drop/rename) | Column-level changes |
| **Automated Refresh** | Data snapshots + schema changes with DML | DDL-only schema changes (column drops) |

See [references/02-sync-behavior.md](references/02-sync-behavior.md) for the complete sync behavior matrix.

## Snowflake Documentation References

- [Use a catalog-linked database](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database)
- [Tutorial: Bidirectional access to Unity Catalog](https://docs.snowflake.com/en/user-guide/tutorials/tables-iceberg-set-up-bidirectional-access-to-unity-catalog)
- [Automatically refresh Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh)
- [Write support for externally managed Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-externally-managed-writes)
- [ALTER DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/alter-database-catalog-linked)
- [Monitor CLDs in Snowsight](https://docs.snowflake.com/en/user-guide/tables-iceberg-monitor-catalog-linked-databases)

## Related Skills in Skillhalla

Several existing CoCo skills cover parts of the Unity Catalog + CLD space. This skill is designed to **complement** them, not replace them.

### Overlapping Skills

| Existing Skill | Repo | What it covers | Overlap with this skill |
|---|---|---|---|
| [unitycatalog-catalog-integration-setup](https://github.com/snowflake-eng/cortex-code-skills/blob/main/data-engineering/iceberg/catalog-integration/unitycatalog-catalog-integration-setup/SKILL.md) | `snowflake-eng/cortex-code-skills` | Catalog integration setup for Unity Catalog (OAuth, bearer token, PrivateLink) | `01-setup-guide.md` (catalog integration portion) |
| [catalog-linked-database](https://github.com/snowflake-eng/cortex-code-skills/blob/main/data-engineering/iceberg/catalog-linked-database/SKILL.md) | `snowflake-eng/cortex-code-skills` | CLD setup, verify, troubleshoot (with sub-skills: create, setup, verify) | `01-setup-guide.md`, `02-sync-behavior.md`, `06-troubleshooting.md` |
| [cld-snowflake-intelligence](https://github.com/snowflake-eng/cortex-code-skills/blob/main/data-engineering/iceberg/cld-snowflake-intelligence/SKILL.md) | `snowflake-eng/cortex-code-skills` | Surfacing CLD tables in Snowflake Intelligence | Not covered here |
| [catalog-linked-db](https://github.com/Snowflake-Solutions/pse-csp/blob/main/AWS/skills/iceberg-Snowflake-on-AWS/catalog-linked-db/SKILL.md) | `Snowflake-Solutions/pse-csp` | CLD creation for AWS Glue REST catalog | `01-setup-guide.md` (partial) |
| [databricks-unity-catalog](https://github.com/snowflake-eng/cortex-code-skills/blob/main/databricks/skills/databricks-unity-catalog/SKILL.md) | `snowflake-eng/cortex-code-skills` | Browse/discover data in Unity Catalog via Databricks CLI | Different angle (Databricks-side exploration) |
| `iceberg` (bundled in CoCo) | Built-in | Required entry point for all Iceberg/CLD work | General router; this skill goes deeper |

### When to Use THIS Skill Instead

Use `unity-catalog-snowflake` when you need:

- **Column drop sync issue** — why drops don't auto-sync but adds do (`03-column-drop-issue.md`)
- **Schema evolution matrix** — which DDL operations sync automatically vs. need manual refresh (`04-schema-evolution.md`)
- **Bidirectional write guide** — DML from Snowflake to Unity Catalog, merge-on-read, partitions (`05-write-support.md`)
- **Monitoring deep-dive** — event table queries, staleness detection, credit tracking (`07-monitoring.md`)
- **Identifier/case resolution** — CASE_INSENSITIVE rules, quoting, DDL normalization (`08-identifier-resolution.md`)
- **Consolidated limitations** — all known gotchas in one place (`09-limitations.md`)

The existing skills are **setup-focused** (getting a CLD running). This skill is **operational** (running, monitoring, troubleshooting, and understanding edge cases after setup).

## Using as a CoCo Skill

This repo doubles as a [Snowflake CoCo](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill. The `SKILL.md` file acts as a router that detects intent and loads the appropriate reference file.

To install:
1. Copy the entire folder to `~/.snowflake/cortex/skills/agent-created/snowflake/unity-catalog-snowflake/`
2. Invoke with `$unity-catalog-snowflake` in CoCo

## Contributing

Found an inaccuracy or want to add coverage for a new topic? PRs welcome. All content should be validated against the [official Snowflake documentation](https://docs.snowflake.com).

## License

MIT
