# Review Before Copying: databricks-lakeflow-connect

## Update Applied

- Updated skill metadata to `0.2.0` and CLI compatibility to `>= v1.0.0`.
- Added 100+ connector catalog positioning and expanded connector examples.
- Rewrote SaaS reference to include file-source, social/ads, dev tool, streaming, and community connector categories.
- Added SharePoint and Google Drive file-source connector guidance.
- Added row filtering guidance with `row_filter: "status = 'ACTIVE' AND created_date >= '2024-01-01'"`.
- Updated scheduling language: multiple custom schedules can be set directly on the pipeline, with Lakeflow Connect creating a Job per schedule.
- Rewrote database connector reference with MySQL integrated CDC beta and query-based connector modes: `SCD_TYPE_1`, `SCD_TYPE_2`, `APPEND_ONLY`.
- Added free-tier planning note: 100 free DBUs/day supporting up to 100M records daily across managed connectors.

## Reviewer Checks

- Confirm exact connector release stages and regional availability before copying to a production skill distribution.
- Confirm whether individual beta connectors require a workspace entitlement or preview channel.
- Re-run a markdown search for stale CLI floors and old scheduling language after copying.
