# SaaS, File Source, Ads, Dev Tool, and Community Connectors

Lakeflow Connect provides managed connectors for SaaS applications, file repositories, advertising platforms, developer tools, streaming systems, and community-built sources. The connector catalog now includes **100+ connectors** across GA, preview, beta, and community categories.

Use this reference for source selection and implementation planning. Always confirm current availability, regional rollout, auth modes, supported objects, and production support level in the per-connector Databricks documentation before creating a production plan.

---

## GA SaaS Connectors Covered by This Skill

| Source | Typical data | Auth | Notes |
|---|---|---|---|
| Salesforce | Accounts, contacts, opportunities, cases, custom objects | OAuth U2M | Use when Salesforce is the system of record for CRM data. |
| Workday Reports | RaaS report output | OAuth refresh token or basic auth | Best for curated Workday reports rather than direct transactional access. |
| ServiceNow | Incidents, changes, assets, CMDB-style tables | OAuth U2M or basic auth | Confirm table ACLs and API rate limits with the ServiceNow admin. |
| Google Analytics 4 | Events and reporting data via BigQuery export | Service-account JSON | Often depends on GA4 export to BigQuery and downstream connector configuration. |
| HubSpot | Contacts, companies, deals, tickets, marketing objects | OAuth | Validate object coverage and incremental fields. |
| Confluence | Spaces, pages, comments, metadata | OAuth | Useful for knowledge/document analytics and search indexing. |

---

## Expanded Connector Catalog

Lakeflow Connect includes many more connectors than the originally documented core set. Examples to consider when users ask about coverage:

| Source | Category | Status / notes |
|---|---|---|
| Aha! | SaaS | Product roadmap and idea-management data. |
| Monday.com | SaaS | Work management boards, items, and updates. |
| Slack | SaaS | Workspace conversations and collaboration metadata where supported by source permissions. |
| Smartsheet | SaaS | Sheets, workspaces, project/task data. |
| Zendesk | SaaS | Tickets, users, organizations, support operations. |
| Zoho Books | SaaS | Accounting, invoices, customers, vendors. |
| Zoom Logs | SaaS | Meeting/webinar/admin log data where available. |
| Veeva Vault | SaaS | Beta as of July 2026; validate access and supported Vault objects. |

---

## File Source Connectors

Use file-source connectors when the source is a managed document repository rather than cloud object storage. Do not confuse these with Auto Loader for S3, ADLS, or GCS paths.

| Source | Status / notes |
|---|---|
| SharePoint | Beta. Use for SharePoint-hosted documents and lists where connector support is enabled. |
| Google Drive | Beta. Use for Drive-hosted documents and metadata where connector support is enabled. |

If the files already land in cloud object storage, prefer Auto Loader through the `databricks-pipelines` skill.

---

## Social and Ads Connectors

| Source | Status / notes |
|---|---|
| Meta Ads | Beta. Validate account permissions, ad object coverage, and API quota. |
| TikTok Ads | Beta. Validate advertiser account access and incremental extraction fields. |
| Google Ads | Beta. Validate customer IDs, manager-account access, and report compatibility. |

---

## Developer Tool Connectors

| Source | Status / notes |
|---|---|
| Jira | Beta. Use for issues, projects, sprints, and agile analytics where enabled. |
| GitHub | Beta. Use for repository, issue, pull request, and workflow analytics where enabled. |

---

## Streaming Connectors

| Source | Status / notes |
|---|---|
| RabbitMQ | Managed streaming ingestion connector. Confirm queue/exchange support, network access, and delivery semantics before production use. |

---

## Community Connectors

Community Connectors are **beta** and open-source. Use them when a managed first-party connector does not exist and the user accepts early-access support trade-offs. Treat community connectors as source templates that may require code review, operational ownership, and additional validation before production use.

---

## Row Filtering

Where supported, apply row filters during both initial load and incremental updates to reduce cost and latency:

```yaml
row_filter: "status = 'ACTIVE' AND created_date >= '2024-01-01'"
```

Use SQL `WHERE`-like predicates on connector-supported fields. Confirm whether the connector pushes the filter to the source API or applies it in the managed ingestion layer.

---

## Implementation Checklist

1. Confirm connector availability, cloud/region support, and release stage.
2. Create or reuse a Unity Catalog `CONNECTION` with the correct auth flow.
3. Confirm source API permissions, rate limits, and incremental extraction fields.
4. Decide ingestion mode, destination catalog/schema/table naming, and row filters.
5. Define one or more pipeline schedules directly on the Lakeflow Connect pipeline.
6. Monitor the pipeline event log and source API quota after the initial load.
