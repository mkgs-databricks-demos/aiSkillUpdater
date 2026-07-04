# Database Connectors

Lakeflow Connect database connectors ingest from operational databases into Unity Catalog Delta tables. Use them when the user wants managed ingestion rather than custom Spark JDBC code.

---

## Connector Families

| Pattern | Sources | Best for | Notes |
|---|---|---|---|
| CDC through gateway | SQL Server cloud/on-prem, PostgreSQL CDC, MySQL CDC depending on rollout | Low-latency change capture from operational systems | Gateway pattern may be required for private networking or source-log capture. |
| Integrated CDC | MySQL integrated CDC beta as of July 1, 2026 | MySQL CDC with one managed pipeline | Single pipeline, no separate ingestion gateway, binlog-based CDC. |
| Query-based ingestion | Oracle, Teradata, SQL Server, PostgreSQL, MySQL, Snowflake, Redshift, Synapse, BigQuery/Federated sources | Sources without usable CDC or when periodic snapshots are enough | Supports `SCD_TYPE_1`, `SCD_TYPE_2`, and `APPEND_ONLY` destination modes. |

---

## SQL Server CDC

Use the SQL Server connector for cloud or on-prem SQL Server sources when users need managed CDC into Delta.

Planning checks:

- Confirm SQL Server edition/version and whether change tracking or CDC is required for the selected connector mode.
- Confirm network path from Databricks to the source, including Private Link, VPN, ExpressRoute, Direct Connect, or public allowlists.
- Create a database user with the minimum privileges required by the connector docs.
- Plan primary keys and destination table names before initial load.
- Query both gateway and ingestion pipeline event logs when troubleshooting.

---

## MySQL Integrated CDC (Beta, July 1, 2026)

MySQL integrated CDC is a beta connector path that uses **one Lakeflow Connect pipeline** and does **not** require a separate ingestion gateway. It captures changes from the MySQL binlog and applies them into Delta tables.

Use this path when:

- The source is MySQL and binlog-based CDC can be enabled.
- The user wants a simpler managed topology than gateway-based CDC.
- Beta feature constraints are acceptable for the workload.

Prerequisites to verify:

- Binary logging is enabled with a compatible row-based format.
- Source retention is long enough to cover outages and backfills.
- The connector user can read required schemas/tables and replication metadata.
- Network access from Databricks to MySQL is configured.

---

## Query-Based Connectors

Query-based connectors run periodic SQL queries instead of reading a native change stream. Use them for sources where CDC is unavailable, expensive to enable, or unnecessary for the user's SLA.

Supported destination modes:

| Mode | Behavior | Use when |
|---|---|---|
| `SCD_TYPE_1` | Upserts latest values by primary key, overwriting previous state. | Consumers only need current records. |
| `SCD_TYPE_2` | Preserves row versions/history with effective intervals or current markers. | Consumers need audit/history for changes. |
| `APPEND_ONLY` | Appends each query result or increment as new rows. | Source emits immutable events or snapshots should be retained. |

Design guidance:

- Include a stable primary key for SCD modes.
- Use a monotonically increasing timestamp or ID for incremental predicates when possible.
- Add row filters to reduce source scan cost:

```yaml
row_filter: "status = 'ACTIVE' AND updated_at >= '2024-01-01'"
```

- Avoid query-based ingestion for very high-churn tables if source-side CDC is available.

---

## Scheduling

Set one or more custom schedules directly on the Lakeflow Connect pipeline. Lakeflow Connect auto-creates a Databricks Job per schedule. Use external Jobs orchestration only when the ingestion must be part of a broader multi-task dependency graph.

---

## Troubleshooting Tips

- Duplicate-key errors usually mean the declared primary key is not unique in the source result set.
- Initial loads can be source-heavy; coordinate with source DBAs and use filters where possible.
- CDC lag often comes from source log retention, network throughput, gateway health, or destination table maintenance.
- Query-based latency is bounded by the schedule plus query runtime; do not present it as true CDC.
