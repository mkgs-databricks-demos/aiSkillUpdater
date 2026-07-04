# Review Before Copying — databricks-zerobus-ingest v0.2.0

This file summarizes all changes made to the skill files relative to the v0.1.0 source at
`~/.databricks/aitools/skills/databricks-zerobus-ingest/`.

---

## SKILL.md

**Version bump:** 0.1.0 → 0.2.0

**Added: REST Interface (Beta) section**
New top-level section documenting the REST API alternative to gRPC:
- Endpoint pattern: `POST /api/2.0/zerobus/ingest/{table_fqn}/records`
- Auth: Bearer token (same as other Databricks REST APIs)
- Use cases: simple HTTP clients, serverless functions, webhooks

**Added: OTLP / OpenTelemetry Integration (Beta) section**
New top-level section documenting OTLP telemetry ingestion:
- Accepts traces, metrics, and logs in OTLP format
- Configure OTEL exporter endpoint to point to Databricks Zerobus
- Automatic OTLP schema → Unity Catalog column mapping

**Updated: Serverless limitation note**
Added sentence: "Use the REST API as an alternative when gRPC is unavailable in serverless environments."

---

## references/1-setup-and-authentication.md

**Added: REST API (Beta) section** (after section 6, before Supported Regions)
Includes a Python `requests` example showing:
- POST to `/api/2.0/zerobus/ingest/{catalog}.{schema}.{table}/records`
- Bearer token authentication
- JSON body format `{"records": [...]}`
- Notes on when to use vs gRPC SDK

**Added: OTLP / OpenTelemetry Integration (Beta) section** (after REST API section)
Includes a Python OpenTelemetry SDK example showing:
- `OTLPSpanExporter` configuration pointing to Databricks Zerobus endpoint
- Bearer token auth header
- Notes on supported signal types (traces, metrics, logs) and schema mapping

---

## references/2-python-client.md

**Added: Arrow Flight (Beta) section** (after Batch Pattern, before Ingestion Method Comparison)
Includes a Python example showing:
- Install: `pip install databricks-zerobus-ingest-sdk[arrow]`
- `ArrowFlightClient` usage with PyArrow `RecordBatch`
- Notes: ~40x throughput vs standard gRPC, PyO3/Rust backend, Python 3.9+ required

**Added: Fire-and-Forget Ingestion Mode section** (after Arrow Flight)
Includes a Python snippet: `client.ingest(records, ack_mode="none")`
Notes on trade-offs: max throughput but no durability guarantee.

**Added: Offset-Based Ingestion Mode section** (after Fire-and-Forget)
Includes a Python snippet showing `start_offset` and `get_committed_offset()`.
Notes on persisting offsets and resume semantics.

---

## references/3-multilanguage-clients.md

**No changes.** Copied verbatim from source.

---

## references/4-protobuf-schema.md

**No changes.** Copied verbatim from source.

---

## references/5-operations-and-limits.md

**Added: System Table Monitoring subsection** (appended to Monitoring and Observability section)
Includes two SQL examples:
- `SELECT * FROM system.lakeflow.zerobus_ingest WHERE table_name = 'my_table' ORDER BY ingest_time DESC;`
- `SELECT * FROM system.lakeflow.zerobus_stream WHERE pipeline_id = 'my_pipeline';`
