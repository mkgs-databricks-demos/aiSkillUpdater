# Review Before Copying

This document summarizes all changes made to the databricks-unity-catalog skill files
in this update batch. Review these before copying files to the source skill directory.

---

## SKILL.md

**Version bump:** 0.1.0 -> 0.2.0

**Added deprecation notice** (immediately after the title/overview):
> Deprecation Notice: `system.workflow.*` tables are deprecated. Use `system.lakeflow.*` instead.

No other content changes. All existing sections, links, and examples preserved unchanged.

---

## references/5-system-tables.md

**Deprecation note added to Lakeflow Schema section header:**
> Deprecation Notice: `system.workflow.*` tables are deprecated. Use `system.lakeflow.*` instead.

**New tables added to the Lakeflow Schema section:**
- `system.lakeflow.job_runs` — aggregated job run records, with example query
- `system.lakeflow.zerobus_ingest` — Zerobus Ingest ingestion records, with example query
- `system.lakeflow.zerobus_stream` — Zerobus streaming state records, with example query

**New section added: `## 2025-2026 System Tables`** (inserted before the External Lineage section)

New tables documented in this section:
- `system.ai.ai_gateway_requests`
- `system.ai.endpoint_usage`
- `system.mlflow.model_versions`
- `system.mlflow.experiments`
- `system.serving.endpoint_usage`
- `system.access.inbound_network_events`
- `system.access.outbound_network_events`
- `system.data_classification.label_history`
- `system.quality_monitor.monitor_run_history`
- `system.quality_monitor.profile_metrics`
- `system.assistant_events`
- `system.clean_room_events`
- `system.marketplace.listing_views`
- `system.marketplace.install_events`
- `system.compute.instance_events`
- `system.compute.instance_pool_events`
- `system.sharing.materialization_history`

Each new table is listed in an overview table. Example SQL queries are provided for
a representative subset of the new tables.

All existing content (Access, Billing, Compute, Query, Information Schema, External
Lineage, Best Practices sections) is preserved unchanged.

---

## references/6-volumes.md

No changes. Copied unchanged from source.

---

## references/7-data-profiling.md

No changes. Copied unchanged from source.
