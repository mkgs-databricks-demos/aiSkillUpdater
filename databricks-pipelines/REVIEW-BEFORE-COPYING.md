# Review Before Copying

This document lists all changes made to the source skill files at
`~/.databricks/aitools/skills/databricks-pipelines/` when producing the
updated versions in this directory.

---

## SKILL.md

**Version bump:** `0.3.0` → `0.2.0`

**Flow and Sink APIs table — new row added** (between Backfill Flow and Sink):

| Feature | Description | Python | SQL |
|---------|-------------|--------|-----|
| Update Flow | Updates an existing flow with new logic; used for joins/aggregations that need to replace specific rows. | `@dp.update_flow()` | `CREATE FLOW name REPLACE WHERE <condition>` |

No other content changed.

---

## references/python-basics.md

**Core decorators list — new entry added** after `@dp.append_flow`:

```
- `@dp.update_flow(target=..., replace_where=...)` — update existing flow rows matching a condition; used for incremental joins/aggregations. See [streaming-table-python.md](streaming-table-python.md).
```

**New section added** at the end of the file:

```markdown
## Update Flow

`@dp.update_flow` updates an existing flow by replacing rows that match a condition. Use this for incremental joins and aggregations where you need to replace specific rows rather than append new ones.

@dp.update_flow("target_table", replace_where="date = current_date()")
def update_daily_aggregates():
    return spark.sql("""
        SELECT date, category, sum(amount) as total
        FROM source_events
        WHERE date = current_date()
        GROUP BY date, category
    """)
```

No other content changed.

---

## references/pipeline-configuration.md

**New section added** between "Multi-Schema Patterns" and "Platform Constraints":

```markdown
## Delta Type Widening

Enable type widening to allow schema evolution with widening type changes (e.g., INT → BIGINT, FLOAT → DOUBLE).
Set in pipeline configuration:

{
  "configuration": {
    "spark.databricks.delta.schema.typeCheck.enabled": "true"
  }
}

Or per-table: add `spark.databricks.delta.schema.autoMerge.enabled = true` to table properties.
Supported widening: INT→LONG, FLOAT→DOUBLE, DATE→TIMESTAMP_NTZ, byte/short→int/long.
```

No other content changed.

---

## All other files

Copied unchanged from source:

- `agents/openai.yaml`
- `assets/databricks.png`
- `assets/databricks.svg`
- `references/1-project-initialization-with-dab.md`
- `references/2-rapid-iteration-with-cli.md`
- `references/auto-cdc-python.md`
- `references/auto-cdc-sql.md`
- `references/auto-loader-python.md`
- `references/auto-loader-sql.md`
- `references/dlt-migration.md`
- `references/expectations-python.md`
- `references/expectations-sql.md`
- `references/foreach-batch-sink-python.md`
- `references/kafka.md`
- `references/materialized-view-python.md`
- `references/materialized-view-sql.md`
- `references/options-avro.md`
- `references/options-csv.md`
- `references/options-json.md`
- `references/options-orc.md`
- `references/options-parquet.md`
- `references/options-text.md`
- `references/options-xml.md`
- `references/performance.md`
- `references/scd-2-querying.md`
- `references/sink-python.md`
- `references/sql-basics.md`
- `references/streaming-patterns.md`
- `references/streaming-table-python.md`
- `references/streaming-table-sql.md`
- `references/temporary-view-python.md`
- `references/temporary-view-sql.md`
- `references/view-sql.md`
