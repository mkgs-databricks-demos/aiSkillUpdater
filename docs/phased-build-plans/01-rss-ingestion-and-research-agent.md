# Build Plan 01 — Phase 2: RSS Ingestion + Research Agent

**Source of truth:** `docs/PROJECT-PLAN.md` §1a, §1b, §1c, §1d, §2 Phase 2,
§3, §5, §6 (#6, #10, #11, #12, #13). This plan restates what's needed to
build Phase 2; it does not re-litigate design decisions already locked
there — if something here seems to conflict with `PROJECT-PLAN.md`, that
file wins and this plan is stale.

**Cross-reviewed:** an independent structural pass (`pi`) and an
independent Databricks-mechanics fact-check pass (`codex`) both reviewed a
first draft of this plan; every BLOCKING finding from both is folded into
the version below. See the changelog at the bottom of this file.

**Why this phase is second (per `PROJECT-PLAN.md` §4's build order):** the
suggested build order runs Phase 2 *before* Phase 1 (grounding corpus)
deliberately — proving the fetch → summarize → cite → classify → propose
loop against one real RSS entry matters more early than having a complete
docs corpus. This plan therefore treats the Research Agent's corpus-lookup
step as **optional and best-effort** (see §5 Part F step 13) rather than a
hard dependency on Build Plan 02, which hasn't been built yet when this
plan is implemented.

---

## 1. Objective

Stand up the event pipeline that turns a new Databricks Release Notes RSS
entry into a reviewable, cited, classified change proposal, with no human
in the loop until the very last step (Phase 3, out of scope here):

1. A scheduled Job polls each of this installation's tracked cloud RSS
   feeds (`tracked_clouds`, from Build Plan 00) daily, using conditional
   GET, and appends new raw items to `rss_bronze`.
2. A Lakeflow Declarative Pipeline dedupes `rss_bronze` into `rss_silver`
   on `guid`, within a bounded time window.
3. A Databricks Jobs **Table update** trigger on `rss_silver` fires the
   Research Agent Job for newly-committed rows.
4. The Research Agent live-fetches each new entry's announcement + linked
   pages, synthesizes a cited summary (`ai_query`), classifies which
   skill(s) it affects (`ai_classify`, multilabel, with a "New Feature"
   fallback bucket), extracts cloud + region-scoped compliance
   availability (`ai_extract`), drafts a proposed skill-file diff per
   affected skill, and writes one JSON findings file per RSS entry to a
   UC Volume.

Out of scope for this plan: the findings-ingestion pipeline that loads
those JSON files into `research_log` and syncs the Vector Search index
(Build Plan 03, Phase 2b), and everything in the Review App (Build Plan
04, Phase 3). This plan's Research Agent only ever writes a JSON file —
it never writes to Delta directly and never touches a skill file.

## 2. Prerequisites

- **Build Plan 00 (Phase 0) deployed and `status = completed` for at least
  one installation** — this plan reads `installations`, `tracked_clouds`,
  and `tracked_regions` (Lakebase, owned by Build Plan 00's schema) and
  needs a working GitHub App connection (to list the configured skills
  repo's current `SKILL.md` files for the classification taxonomy, §5 Part
  F step 15).
- The **`-infra` bundle** already exists (UC catalog/schema, secret scope,
  Lakebase instance, App service principal) — this plan adds new resources
  to that same bundle, per §3 below; it does not create a fourth bundle.
- **The Lakebase schema/migration mechanism chosen in Build Plan 00** must
  be known and reused for this plan's two new tables (§4) — if that choice
  is still unrecorded when this plan starts being built, resolve it in
  Build Plan 00's own doc first (§10 there), don't invent a second
  mechanism here.
- Confirm current DBR default ≥ 17.2 (or whatever DBR version the target
  workspace is pinned to supports Variant Shredding) and that the
  workspace-admin Previews toggle for Variant Shredding is enabled, per
  `PROJECT-PLAN.md` §2 Phase 2's locked assumption — this is a one-time,
  workspace-level admin action, not something this plan's automation can
  self-provision (no documented API for the Previews toggle).
- **Separate, higher DBR/compute requirement for the Research Agent Job
  specifically (Part F only — does not apply to the polling Job, Part
  C)**: `ai_query`, `ai_classify`, and `ai_extract` require **DBR ≥ 18.2**
  and run only on **serverless** compute for notebooks/workflows — they are
  not available on Pro or Classic SQL warehouses. This floor was **verified
  against live Databricks docs during cross-review** (all three functions'
  Requirements sections state it verbatim as of Jun 2026). The
  locally-installed `databricks-dbsql` skill still documents the older
  ~15.1 / 15.4-LTS floor and the legacy v1 API shapes — **do not** downgrade
  this plan to match it; it is stale on the AI Functions surface, a concrete
  instance of exactly the staleness this whole project exists to catch.
  Mechanically, a serverless task selects serverless by **omitting cluster
  config** (per `databricks-jobs`), *not* by pinning a `spark_version`; the
  serverless AI-Functions runtime provides the ≥18.2 requirement, and any
  extra pip deps go in an `environments` block (`client: "4"`). Note this is
  a *higher* floor than Variant Shredding's ≥17.2 above — meeting the AI
  Functions floor automatically satisfies the Variant Shredding one, not the
  other way around.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

| Resource | Bundle | Notes |
|---|---|---|
| `rss_bronze` table DDL (CDF, Row Tracking, `feed_value VARIANT`, Variant Shredding, `CLUSTER BY AUTO`) | `-infra` | Exact DDL in §5 Part A. Created once; never re-created on redeploy (see §7's idempotency note). |
| Service principal — RSS/Research Job | `-infra` | New in this plan (Build Plan 00 only created the App's own SP). One shared SP runs both the polling Job and the Research Agent Job (§5 Part A) — see §10 for why this plan didn't split them into two. |
| `rss_bronze` → `rss_silver` Lakeflow Declarative Pipeline | `-infra` | Per §1d: produces a Delta table `-ai-tools`'s future Vector Search index doesn't directly sync from (only `research_log` does, via Build Plan 03) — `rss_silver` itself is purely internal plumbing, not indexed. |
| RSS polling Job (scheduled, daily) | `-infra` | Writes only to `-infra`-owned tables. |
| Research Agent Job (`trigger.table_update` on `rss_silver`) | `-infra` | **Design call made in this plan** (not explicit in `PROJECT-PLAN.md` §1d): placed in `-infra`, matching the RSS polling Job's rationale — it only *writes* to an `-infra`-owned UC Volume (JSON findings) and *optionally, best-effort* reads the `-ai-tools` docs-corpus Vector Search index (§5 Part F step 13) as a runtime dependency, not a deploy-time one. See §10. |
| JSON findings UC Volume | `-infra` | Created by this plan since the Research Agent is this plan's own deliverable; Build Plan 03 (Phase 2b) only ever *reads* this volume, does not create it. |
| RSS polling cursor + Research Agent watermark tables (Lakebase) | `-infra` (App's Postgres schema) | See §4. Added to the *same* schema Build Plan 00 created, using the same migration mechanism. |

`-ai-tools` is not touched by this plan at all — the docs-corpus Vector
Search index it would eventually host (Build Plan 02) does not exist yet,
and this plan's Research Agent is explicitly built to degrade gracefully
without it (§5 Part F step 13). Do not add a hard dependency on `-ai-tools`
anywhere in this plan's task list.

## 4. Data model

### Delta / Unity Catalog (new in this plan)

**`rss_bronze`** — exact DDL from `PROJECT-PLAN.md` §2 Phase 2, reused
verbatim (do not modify the feature set without re-checking that section):

```sql
CREATE TABLE main.<schema>.rss_bronze (
  feed_cloud STRING,
  guid       STRING,
  fetched_at TIMESTAMP,
  feed_value VARIANT          -- raw <item> as parsed JSON
)
CLUSTER BY AUTO
TBLPROPERTIES (
  delta.enableChangeDataFeed   = true,
  delta.enableRowTracking      = true,
  delta.enableVariantShredding = true
);
```

Note `feed_cloud` is **not** part of the dedup key downstream (`guid` is,
per `<guid>` == `<link>` in these feeds) — it's carried through purely so
`rss_silver`'s transform can report which cloud feed(s) reported a given
`guid`.

**`rss_silver`** (Lakeflow Declarative Pipeline output — streaming table,
DLT-managed, reads `STREAM(rss_bronze)`):

| Column | Type | Notes |
|---|---|---|
| `guid` | STRING | Dedup key |
| `title` | STRING | |
| `link` | STRING | Identical to `guid` in these feeds; kept as a separate column for readability in downstream consumers |
| `pub_date` | TIMESTAMP | Day-granular (`00:00:00 GMT`) per all 3 feeds |
| `description` | STRING | Raw HTML fragment from CDATA — **not** stripped to plain text here; the Research Agent (or a later consumer) parses it if plain text is needed |
| `categories` | ARRAY<STRING> | All repeated `<category>` values, not just the first |
| `source_feed_clouds` | ARRAY<STRING> | Which cloud feed(s) reported this `guid` within the dedup window — populated by merging bronze rows that share a `guid`, not just keeping the first-seen one, since a cross-cloud announcement legitimately has cloud-specific availability nuance the Research Agent may want later |
| `first_seen_at` | TIMESTAMP | Earliest `fetched_at` among the merged bronze rows for this `guid` |

**Dedup mechanism — two-flow design, primary (not a fallback)**: a single
unwindowed `groupBy(guid).agg(...)` cannot coexist with
`dropDuplicatesWithinWatermark` in one Lakeflow Declarative Pipelines flow
— `dropDuplicatesWithinWatermark` is specifically designed to *drop* later
duplicate rows, not merge their column values, and Databricks' own
streaming-aggregation guidance uses watermarked *windowed* aggregations,
not an unwindowed per-key `groupBy` emitting one final append row
(cross-review finding — the single-flow approach in an earlier draft of
this plan was unrealistic). Use two chained flows instead:
1. **Dedup flow**: `STREAM(rss_bronze)` → `dropDuplicatesWithinWatermark`
   on `guid` with `withWatermark("pub_date", "3 days")` (a 3-day window is
   generous given all cross-cloud publishes of the same `guid` are
   expected to land within the same daily poll cycle or the next one) →
   writes a first-seen-only version of `rss_silver` (all columns above
   except `source_feed_clouds` populated from whichever bronze row won
   the dedup).
2. **Backfill flow**: a small batch/micro-batch step, triggered after
   flow 1 within the same pipeline run, that re-scans `rss_bronze` for
   every `guid` written by flow 1 in this run (bounded to that run's new
   rows, not a full table scan) and updates that row's
   `source_feed_clouds` (all distinct `feed_cloud` values seen for that
   `guid` within the window) and `first_seen_at` (earliest `fetched_at`
   among them). This is a plain bounded read + targeted update, not a
   second unbounded streaming aggregation.
Do not silently drop the cross-cloud info by skipping step 2 — that's a
real design requirement, not a nice-to-have.

**`rss_silver` must itself declare CDF + Row Tracking** (via the pipeline's
table-definition `table_properties`, e.g.
`table_properties={"delta.enableChangeDataFeed": "true", "delta.enableRowTracking": "true"}`
on the `rss_silver` table definition) — `TABLE_CHANGES`/CDF-based reads in
§5 Part F step 11 require CDF to actually be enabled on `rss_silver`
itself; it is **not** inherited automatically from `rss_bronze` having it.
Confirm the exact current Lakeflow Declarative Pipelines syntax for
setting Delta table properties on a streaming-table definition before
implementing (field/decorator names may differ from a plain `CREATE
TABLE`'s `TBLPROPERTIES`).

**JSON findings file schema** (one file per `rss_silver` row, written by
the Research Agent to the UC Volume — **this is a contract Build Plan 03
must match exactly**; record here as the authoritative version):

```json
{
  "schema_version": "1",
  "installation_id": "<uuid, from Lakebase installations>",
  "research_agent_run_id": "<uuid, one per Research Agent Job run>",
  "rss_entry": {
    "guid": "...",
    "link": "...",
    "title": "...",
    "pub_date": "2026-07-01T00:00:00Z",
    "source_feed_clouds": ["aws", "azure"]
  },
  "processed_at": "2026-07-06T18:00:00Z",
  "summary": "<markdown, cited inline as [n]>",
  "citations": [
    {"id": 1, "url": "https://docs.databricks.com/...", "title": "..."}
  ],
  "skill_classifications": [
    {
      "label": "databricks-pipelines",
      "is_new_feature_bucket": false,
      "target_skill_path": "databricks-pipelines/SKILL.md",
      "availability": {
        "clouds": ["aws", "gcp"],
        "compliance": {
          "no_csp":            {"us-west-2": "true"},
          "hipaa":              {"us-west-2": "unknown"},
          "pci_dss":            {"us-west-2": "false"},
          "fedramp_moderate":   {"us-west-2": "false"}
        }
      },
      "proposed_diff": "<unified diff text, or null if not yet drafted>"
    }
  ],
  "status": "pending_review",
  "errors": []
}
```

Design notes on this schema (record decisions, don't leave them implicit):

- **Compliance tri-state is modeled as a string enum (`"true"`/`"false"`/
  `"unknown"`), not a JSON boolean mixed with a string** — matches
  `PROJECT-PLAN.md` §1b's `ai_extract`-schema guidance (tri-state needs an
  enum, not a native boolean) and avoids Autoloader inferring an
  inconsistent column type across files where some entries have `unknown`
  and others have `true`/`false`. Compliance keys are only present for
  this installation's **tracked regions** (`tracked_regions`, from Build
  Plan 00) — never every region Databricks supports.
- `availability.compliance.fedramp_moderate` is `"false"` (never
  `"unknown"`) for any tracked region outside the FedRAMP Moderate
  authorization boundary — per §1b, that boundary is a small,
  documented, fixed set of `us-*` AWS regions (4 as of `PROJECT-PLAN.md`'s
  writing), definitive rather than a judgment call. **This plan does not
  hardcode the specific region list** — the Research Agent's step 16
  fetch of the live FedRAMP Moderate page (§5 Part F step 16) is exactly
  what determines the current boundary each time it runs, so a future
  change to that boundary is picked up automatically rather than needing
  a code change here. An implementer building this from scratch (without
  also reading `PROJECT-PLAN.md` §1b/§6 #11) should fetch
  `docs.databricks.com/aws/en/security/privacy/fedramp` directly to see
  the current authorized-region list before writing the boundary-check
  logic — do not guess or hardcode a list from memory.
- **`no_csp` availability's source, not stated elsewhere in this file**:
  unlike the three named compliance standards, "No CSP" (standard,
  non-CSP workspace) availability comes from the main CSP/security-profile
  overview page — `docs.databricks.com/aws/en/security/privacy/
  security-profile` (§1b) — which also carries the cross-feature
  preview/beta support table. Treat a feature as `no_csp: "true"` for a
  tracked region once the live-fetched announcement/docs (step 12/13)
  confirm general availability in that region, `"unknown"` if GA status
  in that specific region isn't confirmed by any fetched source, and only
  ever `"false"` if a source explicitly states the feature is unavailable
  there.
- `target_skill_path` is `null` when `label` is `"New Feature"` or a
  user-created bucket with no existing skill file yet (Build Plan 04's
  override flow handles turning that into an actual new skill later) —
  `proposed_diff` should still be populated with drafted *content* for a
  brand-new file in that case, just not as a "diff against an existing
  file."
- `errors` is populated (non-fatal) when a *specific* step failed for
  this entry (e.g. one classification's `ai_extract` call errored) —
  per §7's non-throwing execution requirement, a partial failure writes
  what succeeded plus an error entry, rather than dropping the whole RSS
  entry or crashing the Job.
- `schema_version` exists from day one so Build Plan 03's Autoloader
  pipeline can branch on it later without a breaking change if this
  shape needs to evolve.

### Lakebase (new in this plan, same schema/migration mechanism as Build Plan 00)

**`rss_polling_cursor`**

| Column | Type | Notes |
|---|---|---|
| `installation_id` | FK | |
| `cloud` | enum(`aws`, `gcp`, `azure`) | |
| `etag` | text, nullable | From the last successful fetch's response headers; null on first-ever poll for this feed |
| `last_modified` | text, nullable | Fallback header for servers that don't honor `If-None-Match` |
| `last_checked_at` | timestamptz | Updated on every run regardless of `200`/`304` — useful for an "is polling healthy" observability check |
| `last_change_at` | timestamptz, nullable | Updated only on a `200` (i.e. an actual content change), separate from `last_checked_at` |

PK: `(installation_id, cloud)`. Only rows for clouds in this
installation's `tracked_clouds` should exist — the polling Job should not
create a cursor row for a cloud the installation doesn't track, and
should clean up (or at minimum ignore) a cursor row for a cloud the user
later un-tracks.

**`rss_silver_watermark`**

| Column | Type | Notes |
|---|---|---|
| `installation_id` | PK | One row per installation |
| `last_processed_commit_version` | bigint, nullable | The `rss_silver` Delta table commit version through which the Research Agent has already enumerated rows; null before the first successful run |
| `updated_at` | timestamptz | |

This is the "idempotent processed/seen marker" `PROJECT-PLAN.md` §2 Phase
2 requires, since the `trigger.table_update` trigger fires on commit, not
per row (§5 Part E). Using a single watermark version (rather than a set
of already-seen `guid`s) keeps this table's write pattern a plain
single-row upsert per run, consistent with everything else's Lakebase
profile. Keep it a single-statement upsert — do **not** wrap the watermark
advance plus findings writes in a SQL multi-statement transaction; per
`databricks-dbsql` `references/sql-scripting.md`, multi-statement
transactions / `BEGIN ATOMIC` are Preview and require `catalogManaged`,
which this design avoids. The ordering guarantee (advance the watermark
only after all findings files are written — Part F step 20) is enforced by
task sequencing, not a DB transaction.

## 5. Task breakdown

### Part A — `rss_bronze` table + service principal (`-infra`)

1. Add the `rss_bronze` `CREATE TABLE` DDL (§4) to the `-infra` bundle.
   **Idempotency**: if a redeploy re-runs table creation, use
   `CREATE TABLE IF NOT EXISTS` — never `CREATE OR REPLACE`, which would
   silently drop existing data and history on every redeploy. Whatever
   DAB resource type or setup-job pattern the bundle uses for table DDL
   (per `databricks-dabs` guidance already cross-reviewed into Build Plan
   00), reuse the same pattern here rather than introducing a second one.
2. Create one new service principal for this plan's Job(s) — the RSS
   polling Job and the Research Agent Job share this one identity (see
   §10 for why this plan didn't create two). Grant it `MODIFY` + `SELECT`
   on `rss_bronze`, `SELECT` on `rss_silver` (once it exists, Part D),
   read/write on the JSON findings UC Volume (Part B below), and read
   access to the Lakebase schema's `rss_polling_cursor` and
   `rss_silver_watermark` tables plus read-only access to
   `installations`/`tracked_clouds`/`tracked_regions` (needs to read
   Build Plan 00's tables, never write them).
3. This SP needs read access to the configured skills repo — the
   Research Agent (Part F step 15) needs to list current `SKILL.md`
   files to build the classification taxonomy — but **which of two
   designs gets this SP that access is genuinely undecided, not just
   unconfirmed**: (a) grant this SP direct use of Build Plan 00's GitHub
   App installation token flow (§1a) so it mints a token itself, or (b)
   have the Research Agent Job call back into the App (or a shared
   internal library/endpoint) to obtain a token, if minting turns out to
   be scoped to the App's own runtime identity only. Build Plan 00's own
   doc does not itself grant the Research Agent SP secret access or
   define an internal token-mint endpoint, so this cannot be resolved by
   re-reading that plan alone — it needs a decision made when Build Plan
   00 is actually implemented. **Resolve this before Part F step 15 is
   built**; don't assume either answer without checking.

### Part B — JSON findings UC Volume (`-infra`)

4. Create the UC Volume the Research Agent writes JSON files to (e.g.
   `main.<schema>.research_findings`). Apply explicit `grants`
   (`READ_VOLUME` for Build Plan 03's future pipeline reader, `WRITE_VOLUME`
   for this plan's Research Agent SP) — per `databricks-dabs` guidance
   already cross-reviewed into this project (a generic bundle
   `permissions` block does not cover Volume access).
5. Decide and document a **file-naming/layout convention** now, since
   Build Plan 03's Autoloader will glob this path: e.g.
   `/Volumes/main/<schema>/research_findings/<installation_id>/<rss_guid_hash>.json`
   or a flat `<installation_id>__<uuid>.json` — either is fine, but
   record the actual choice here once made (see §10), since Build Plan 03
   needs it to write its Autoloader source path/pattern correctly.

### Part C — Daily polling Job task (`-infra`)

6. Implement the Job task exactly as specified in `PROJECT-PLAN.md` §2
   Phase 2 (verbatim steps, restated here so this plan is self-contained):
   1. **Load the cursor** — for each cloud in this installation's
      `tracked_clouds` (Build Plan 00), read `rss_polling_cursor`
      (`etag`/`last_modified`); absent row means first-ever poll for that
      feed (unconditional GET).
   2. **Conditional GET** — `If-None-Match: <etag>` and/or
      `If-Modified-Since: <last_modified>` against that cloud's feed URL
      (AWS `docs.databricks.com/aws/en/feed.xml`, GCP
      `docs.databricks.com/gcp/en/feed.xml`, Azure
      `learn.microsoft.com/en-us/azure/databricks/feed.xml`). A `304`
      means no change — skip to step 5 for this feed. A `200` means parse.
   3. **Parse** — RSS 2.0 XML → item records: `title`, `link`, `guid`
      (dedup key), `pub_date`, `description` (HTML in CDATA, kept raw),
      `category` (repeatable, collect as a list). Stamp each record with
      `feed_cloud` and the run's `fetched_at`.
   4. **Append to bronze** — one plain
      `.write.format("delta").mode("append")` per run (not per-item), the
      raw parsed item as JSON going into `feed_value` VARIANT.
   5. **Save the cursor** — upsert `rss_polling_cursor` regardless of
      `200`/`304` (always update `last_checked_at`; update `etag`/
      `last_modified`/`last_change_at` only on `200`).
   6. Repeat 1–5 independently per tracked cloud within the same task run
      — a `304` on one feed and a `200` on another in the same run is
      normal, not an error.
   Runs on **serverless compute** (no compute-tier requirement for this
   workload; per `databricks-jobs`, serverless is selected by omitting
   cluster config, **not** by pinning a DBR version). Schedule **daily**
   via a `schedule` block with all three fields `databricks-jobs` requires
   together — `quartz_cron_expression`, `timezone_id`, and
   `pause_status: UNPAUSED` (a missing/invalid timezone or a paused status
   is a listed "schedule not triggering" cause) — e.g.
   `quartz_cron_expression: "0 0 18 * * ?"`, `timezone_id: "UTC"`,
   `pause_status: UNPAUSED`. An afternoon/evening-UTC hour aligns with
   typical `<lastBuildDate>` updates (per §2 Phase 2); pick the specific
   hour and record it in §10 once chosen.
7. **Multi-installation note**: if this Job is meant to serve every
   installation from one deployed instance (vs. one Job per installation
   — confirm which model this deployment actually uses, since
   `PROJECT-PLAN.md`'s intro assumes multi-tenant capability but a given
   deployment may in practice run single-tenant), step 1 must loop over
   *all* installations' `tracked_clouds`, not just a single hardcoded one.
   Record which model was actually built here once decided (see §10). If
   the one-Job-serves-N-installations model is chosen, `databricks-jobs`'s
   native `for_each_task` (loop over inputs with a `concurrency` cap,
   inputs supplied from an upstream task via `dbutils.jobs.taskValues`) is
   the idiomatic construct for the per-installation loop rather than
   hand-rolled iteration.

### Part D — `rss_bronze` → `rss_silver` pipeline (`-infra`)

8. Build the two chained Lakeflow Declarative Pipeline flows per §4's
   `rss_silver` schema and dedup mechanism: the dedup flow
   (`STREAM(rss_bronze)` → `dropDuplicatesWithinWatermark` on `guid` with
   a `pub_date` watermark) and the backfill flow (bounded re-scan of
   `rss_bronze` to populate `source_feed_clouds`/`first_seen_at` for that
   run's newly-written rows). Confirm the exact current Lakeflow
   Declarative Pipelines syntax for chaining a batch step after a
   streaming flow within one pipeline before implementing — the two-flow
   shape itself is the settled design (§4), not something to second-guess
   back into a single unwindowed `groupBy`.
9. This pipeline is the sole DLT-managed writer of `rss_silver` — nothing
   else ever writes to it directly (consistent with `rss_bronze` being
   the sole *reader* source, per the earlier LDP-vs-`http_request`
   rejection already locked in `PROJECT-PLAN.md`). Declare `rss_silver`
   with `CLUSTER BY AUTO` as well — `databricks-dbsql`
   `references/best-practices.md` recommends liquid clustering by default
   for new Delta tables including streaming tables — and it needs CDF +
   Row Tracking enabled on itself (§4) for Part F step 11's `table_changes`
   read to work at all (those are not inherited from `rss_bronze`).

### Part E — Table update trigger (`-infra`)

10. Configure the Research Agent Job's trigger:
    ```yaml
    trigger:
      pause_status: UNPAUSED           # REQUIRED — sibling of table_update;
                                       # omitting it is databricks-jobs'
                                       # #1 listed "trigger not firing" cause
      table_update:
        table_names: ["main.<schema>.rss_silver"]
        condition: ANY_UPDATED         # only value databricks-jobs documents;
                                       # do NOT use ALL_UPDATED without
                                       # verifying it exists in the live SDK
        min_time_between_triggers_seconds: 600
        wait_after_last_change_seconds: 120
    ```
    per `PROJECT-PLAN.md` §2 Phase 2 / §6 #13 — a real, UC-table-scoped
    native Jobs trigger type, distinct from `file_arrival` (Volumes/
    external locations only, never managed tables). The `pause_status`
    and both debounce fields above are documented for `table_update` in
    `databricks-jobs` `references/triggers-schedules.md`
    (`wait_after_last_change_seconds` lets several `rss_silver` commits
    from one pipeline run coalesce into a single Research Agent run rather
    than firing per-commit). Separately, on the Research Agent **Job**
    (not the trigger block), set `max_concurrent_runs: 1` with
    `queue: {enabled: true}`: per `databricks-jobs`, the default
    `max_concurrent_runs: 1` with no queue silently **skips** any trigger
    that fires while a run is active; enabling the queue runs them
    sequentially instead (the watermark makes a dropped fire self-healing
    only if a *later* commit ever arrives, so queueing is the safer
    default). The `table_update` trigger exposes **no** payload (no commit
    version/timestamp) — see step 11 for how this run's version bound is
    obtained instead.

### Part F — Research Agent Job (`-infra`)

11. On trigger fire, first **enumerate the actually-new rows** — do not
    assume one trigger firing maps to one new row. Read
    `rss_silver_watermark.last_processed_commit_version` for this
    installation (call it `start_version`); separately capture an
    explicit `end_version` for this run as the table's latest committed
    version at the moment this step runs (e.g. via `DESCRIBE HISTORY` or
    an equivalent version-lookup call), captured *before* any further
    processing. The `table_update` trigger exposes **no** payload (no
    commit version or timestamp) — `databricks-jobs`
    `references/triggers-schedules.md` documents a trigger payload only
    for `file_arrival`, never `table_update` — so this version lookup is
    the only supported way to bound the run, not a fallback to a
    payload value. Then call
    `table_changes('main.<schema>.rss_silver', start_version + 1,
    end_version)` — an explicit bounded `[start, end]` range, **not**
    an open-ended "start to latest" call — so that any `rss_silver` commit
    that lands *after* this run starts is deliberately excluded from this
    run and picked up cleanly by the next one, rather than being
    nondeterministically included or excluded depending on timing.
    `table_changes` requires CDF to actually be enabled on `rss_silver`
    (see §4's new note on this) — this will fail loudly, not silently
    return nothing, if that property is missing. Process each row in
    the resulting change set as one RSS entry through steps 12–18 below;
    only advance `rss_silver_watermark.last_processed_commit_version` to
    this run's `end_version` after **all** entries in this batch have
    been written to the findings Volume (partial-batch failure should not
    silently skip re-processing on retry — see §7).
12. For each new `rss_silver` row, **live-fetch** the announcement's
    `link` plus every URL referenced in its `description`/body — never
    rely solely on a cached corpus, since same-day announcements often
    link to pages not yet indexed anywhere.
13. **Optional, best-effort**: if the `-ai-tools` docs-corpus Vector
    Search index (Build Plan 02) exists, query it for supplementary
    context on the subject. **If it does not exist yet (expected, since
    this plan is built before Build Plan 02 per §4's stated build order),
    this step must no-op cleanly** — check for the index's existence (or
    catch the specific "index not found" error) and proceed without it,
    never fail the whole Research Agent run over a missing corpus. Do
    not build a placeholder corpus just to satisfy this step; that's
    explicitly out of scope per the Objective.
14. **Synthesize with `ai_query`** (not `ai_summarize` — needs
    cross-document reasoning over multiple fetched URLs with a citation
    contract, per `PROJECT-PLAN.md`'s cross-review against
    `databricks-ai-functions`) over the live-fetched content (+ step 13's
    context if present) into: a markdown summary with inline `[n]`-style
    citation markers, and a parallel `citations` array of
    `{id, url, title}`. Structure the prompt/output contract so this is
    parseable into the JSON schema's `summary`/`citations` fields
    directly, not free-text requiring a second parse. If relying on
    `ai_query`'s structured `responseFormat`, note (per `databricks-dbsql`
    `references/ai-functions.md`) that a top-level `responseFormat` STRUCT
    must contain **exactly one** field — wrap the summary+citations pair in
    a single top-level object (e.g. `STRUCT<result: STRUCT<summary ...,
    citations ...>>`) rather than two top-level fields.
15. **Classify with `ai_classify`**, explicitly pinned to
    `options => map('version', '2.0', 'multilabel', 'true', 'instructions', <taxonomy context>)`
    — the multilabel behavior, per-label descriptions, and up to
    500-label capacity this plan relies on are v2-only (verified against
    live docs during cross-review — v2 returns an array with up to 500
    labels; the `databricks-dbsql` skill still documents only the legacy
    v1 single-label / 20-label `STRING` API and is stale here); the older
    default/v1 behavior has materially smaller limits and no multilabel
    mode. Build the label set fresh each run from the configured skills
    repo's current `SKILL.md` files (name + one-line description each),
    **plus** an always-present `"New Feature"` label, **plus** any
    user-created bucket names already recorded from prior overrides
    (Build Plan 04 — if that table doesn't exist yet when this plan is
    built, the taxonomy is just skills + "New Feature" for now; extend
    once Build Plan 04 lands). Guardrails (cross-reviewed limits already
    locked in `PROJECT-PLAN.md` §2 Phase 2): keep each label ≤100 chars
    and description ≤1000 chars, keep the taxonomy well under the
    documented 500-label ceiling, pass extended classification context
    via the function's `instructions` option rather than inflating the
    label list itself. `ai_classify` returns **no native
    confidence/rationale** — do not populate those fields in the JSON
    schema (they aren't in it above, deliberately); if the Review App
    (Build Plan 04) wants them later, that's a separate small LLM call,
    not something to retrofit here.
16. **Extract availability with `ai_extract`**, scoped to this
    installation's `tracked_regions` (never every region Databricks
    supports): cloud availability (AWS/Azure/GCP) from the fetched
    sources (step 12/13), plus compliance availability against the three
    fixed "Regional support for features" pages (`docs.databricks.com/
    aws/en/security/privacy/{hipaa,pci,fedramp}` — this is a periodic
    reference fetch against fixed URLs, independent of what the
    announcement itself links to, per §1b), plus `no_csp` from the
    security-profile overview page (see §4's new note on its source) —
    all for each tracked region. Model the tri-state as a string enum in
    the extraction schema, not a boolean (§1b's `ai_extract`-schema
    guidance); keep the schema within documented limits (≤128 fields,
    ≤7 nesting levels) by scoping to just this installation's tracked
    clouds/regions. **Pin `options => map('version', '2.0')` explicitly**
    — current `ai_extract` v2.1 (now the recommended default, verified live)
    changes the return shape so each scalar
    field comes back as an object (`{value, citation_ids,
    confidence_score}`) rather than the raw value, which would require
    every `result:response:<field>` access in this plan to become
    `result:response:<field>:value` instead. Pinning `2.0` keeps
    `result:response:...` returning the plain scalar directly, matching
    every access pattern described in this plan and in `PROJECT-PLAN.md`
    §1b. If a future revision of this plan moves to v2.1 for its
    per-field confidence scores, every `result:response:...` reference in
    this file must be updated to the `:value` form at the same time —
    don't mix versions. `fedramp_moderate` is only ever assessed for
    AWS-tracked regions (§1b) — skip the call entirely for non-AWS
    regions rather than asking the model and getting a meaningless
    answer.
    **`ai_prep_search` considered, not used here (investigated,
    two independent lenses)** — it's a real Beta AI Function
    (`ai_prep_search(parsed, [options])`) that chunks and
    context-enriches text for embedding, but its hard input contract is
    **`ai_parse_document`'s VARIANT output only** (binary document files:
    PDF/DOC/DOCX/PPT/PPTX/images). This step's inputs (live-fetched HTML
    announcement pages, the `research_log` row itself) are neither
    binary documents nor `ai_parse_document` output, so `ai_prep_search`
    doesn't apply to the Research Agent or to `research_log`'s own
    Vector Search index — each `research_log` row stays the retrieval
    unit, prepared with plain string concatenation (title/summary/
    classification/skill metadata), not `ai_prep_search`. See
    `PROJECT-PLAN.md` §"Phase 1" for the one place `ai_prep_search`
    could genuinely fit (a future document-file-based docs corpus).
17. **Draft a proposed diff per classified skill/bucket** — a unified
    diff against the skill's current `SKILL.md`/references content
    (fetched via the same GitHub App token as step 15/Part A step 3) for
    an existing skill, or drafted new content (no diff, since there's no
    base file) for `"New Feature"`/a new bucket. Ground the draft in the
    step 14 summary + citations + step 16 availability findings — never
    invent claims not traceable to a fetched source.
18. **Write the JSON findings file** (§4 schema) to the Volume (Part B),
    one file per `rss_silver` row processed this run, `status:
    "pending_review"`.
19. **Non-throwing execution for steps 14–17**: capture errors per AI
    Function call rather than aborting the whole entry — the mechanism is
    **function-specific** (verified live during cross-review): `ai_query`
    takes a `failOnError => false` argument and returns
    `STRUCT<result, errorMessage>`; `ai_classify` and `ai_extract` do
    **not** accept `failOnError` — instead they already return a VARIANT
    carrying an `error_message` field (null on success), so read that field
    rather than passing an unsupported argument. Either way, a failed step
    for one entry writes what succeeded plus an `errors` entry (§4 schema)
    into that entry's JSON file, rather than dropping the RSS entry or
    crashing the whole batch. A Phase 7 alert
    on non-empty `errors` arrays is out of scope for this plan (Build Plan
    07) but the `errors` field must exist and be populated correctly now
    so that later plan has something to alert on.
20. Only after every entry in this run's batch has a findings file
    written (successfully or with a captured `errors` entry — "processed"
    ≠ "error-free"), advance `rss_silver_watermark.last_processed_commit_version`
    for this installation (per step 11's ordering requirement).

## 6. Explicit fan-out map

**Preconditions (block everything else in this plan):**
- Build Plan 00 must be deployed and have at least one `completed`
  installation before *any* of this plan's tasks can be meaningfully
  tested end-to-end (Part C step 7 reads `tracked_clouds`; Part F step 15
  reads the configured repo via the GitHub App token flow).
- The Lakebase migration mechanism and the "one Job serves N installations
  vs. one Job per installation" model (§5 Part C step 7) must both be
  decided before Part C/Part F's implementation can be scoped precisely
  — treat "confirm both against Build Plan 00's actual state" as step 0
  of this plan.

**Can run in parallel once preconditions are met:**
- Part A (table + SP + repo-token-access grants), Part B (Volume + naming
  convention), and the Lakebase `rss_polling_cursor`/
  `rss_silver_watermark` table creation (§4) touch entirely separate
  bundle resources and Lakebase schema objects for their *creation* work
  — three independent sub-agent tasks can draft each in parallel, the
  same coordination Build Plan 00's §6 uses for concurrent
  `databricks.yml` edits. **However, mirroring Build Plan 00's own
  caveat**: Part A step 2's *grants* (`SELECT` on `rss_silver`, read/write
  on the findings Volume, read on the two new Lakebase tables) target
  objects Part D, Part B, and the Lakebase-table task respectively
  create — so Part A can be drafted in parallel with the other two, but
  its grants cannot be *applied and verified* until those objects
  actually exist. Don't mark Part A "done" independent of that
  dependency; only its table-DDL/SP-creation scaffolding is truly
  parallel. Coordinate concurrent `databricks.yml`/schema-migration file
  edits the same way (serialize edits to the shared
  bundle file, or have one task own it and the others submit patches).
- Within Part F, steps 14 (`ai_query` synthesis), 15 (`ai_classify`), and
  16 (`ai_extract`) are **not** independent of each other in the sense of
  "build with zero shared context" — 17's diff-drafting step consumes all
  three outputs, and 15's taxonomy needs to exist before 17 knows which
  skill file(s) to draft against — but the three AI Function *call sites*
  themselves (prompt/schema design, guardrail implementation, error
  capture) can be built and unit-tested independently against fixture
  input (a saved fetched-page + a saved RSS entry) before the live-fetch
  plumbing (step 12) is wired up. Treat "build the fetch pipeline" (steps
  11–13) and "build the three AI Function call sites against fixtures"
  (steps 14–16, tested standalone) as two parallelizable tracks that
  integrate at step 17.

**Must be sequential:**
- Part D (pipeline) depends on Part A (Part D reads `rss_bronze`).
- Part E (trigger) depends on Part D (`rss_silver` must exist to be a
  trigger target) and on Part F's Job existing (the trigger attaches to
  the Research Agent Job's config) — build the Research Agent Job's
  compute/task definition (without the trigger wired yet) before Part E,
  then attach the trigger last.
- Part F step 11 (watermark-based enumeration) must be built and tested
  before steps 12–20 are meaningfully exercisable end-to-end — an
  incorrect watermark implementation either reprocesses old entries
  forever or silently skips new ones, and either failure mode is easy to
  miss in a manual single-entry test (§8's validation checklist calls
  for a specific multi-run test to catch this).
- Part C (polling Job) must run at least once successfully (producing
  real `rss_bronze`/`rss_silver` rows) before Part F can be tested against
  anything other than manually-inserted fixture rows.

**Cross-plan fan-out:** this plan's JSON findings schema (§4) is a shared
contract with Build Plan 03 (Phase 2b) — whoever builds 03 must match it
exactly, including the string-enum tri-state choice and the file
naming/layout convention from Part B step 5. This plan and Build Plan 02
(grounding corpus) can be built fully in parallel (§13's optional
Vector-Search read is designed to no-op without it), sharing only the
`-infra` bundle file (same coordination note as Build Plan 00's §6).

## 7. Platform constraints to build against (cross-review-confirmed in `PROJECT-PLAN.md`)

- **Why a plain Job, not a Lakeflow Declarative Pipeline `http_request`
  dataset, does the polling** (already investigated and locked in
  `PROJECT-PLAN.md` §2 Phase 2): `http_request`'s return type is only
  `STRUCT<status_code INT, text STRING>` — it exposes **no response
  headers**, so `ETag`/`Last-Modified` can't be read back for conditional
  GET (corroborated by `databricks-dbsql` `references/ai-functions.md`); it
  also requires a pre-provisioned UC HTTP `CONNECTION` object plus
  `USE CONNECTION` privilege (setup this design otherwise doesn't need);
  and LDP's retry/full-refresh semantics fight a side-effecting, stateful
  HTTP call regardless of which function makes it. (The earlier "scoped
  away from high-volume batch use" phrasing is dropped — `databricks-dbsql`
  does not state that limitation; the rejection stands on the header,
  connection, and LDP-semantics reasons above.) Don't revisit this
  without a genuinely new reason — it's a settled, evidenced decision.
- **`read_files` is also not a live-polling mechanism** (per
  `databricks-dbsql` `references/ai-functions.md`): it reads files already
  in cloud storage or a UC Volume (XML/glob/streaming supported) — it does
  **not** perform live HTTP fetches or expose response headers/cursors, so
  it can't replace the polling Job's conditional GET. It would be relevant
  only if the polling Job first landed raw RSS/XML snapshots to a Volume,
  after which `read_files(..., format => 'xml')` could parse them — a
  strictly more complex two-hop path with no benefit here, so not used.
- **Job-level retries + failure notifications** (per `databricks-jobs`
  `references/notifications-monitoring.md`): set `max_retries` on **both**
  Jobs — both are designed idempotent (the polling Job's conditional-GET +
  cursor; the Research Agent's watermark advances only after all findings
  are written), so a task-level retry safely covers transient
  fetch/`ai_query` failures. Also set `email_notifications.on_failure`
  (and/or `webhook_notifications.on_failure`) on both — this is the skill's
  idiomatic health signal and is what §8's "is polling healthy" check and
  Build Plan 07's future error alerting hook into.
- **The Table update trigger fires on commit, not per row** — Part F
  step 11's watermark-based enumeration exists specifically to handle
  this; don't build Part F assuming "one trigger fire = one new
  `rss_silver` row."
- **Variant Shredding is Beta** and needs both a DBR floor (≥17.2 as of
  this writing) and a workspace-admin Previews toggle enabled once per
  installation/workspace — confirmed in §2's Prerequisites; if a
  workspace hasn't enabled it, Part A step 1's `CREATE TABLE` will fail,
  not silently skip the feature.
- **AI Functions need a higher DBR/compute floor than Variant Shredding**
  (§2 Prerequisites): `ai_query`/`ai_classify`/`ai_extract` require
  DBR ≥18.2 and serverless compute (no Pro/Classic SQL warehouse support),
  **verified live during cross-review** — select serverless by omitting
  cluster config (not by pinning a DBR); applies only to the Research Agent
  Job (Part F), not the polling Job (Part C).
- **`ai_classify` guardrails** (§5 Part F step 15): pin
  `options => map('version', '2.0', 'multilabel', 'true', ...)` — v1/
  default behavior lacks multilabel mode and has much smaller limits.
  ≤100 char labels, ≤1000 char descriptions, taxonomy well under the
  500-label ceiling, no native confidence/rationale output.
- **`ai_extract` guardrails** (§5 Part F step 16): pin
  `options => map('version', '2.0')` — v2.1 wraps each scalar field in an
  object (`{value, citation_ids, confidence_score}`), which changes every
  `result:response:...` access in this plan to need a `:value` suffix;
  staying on 2.0 keeps direct scalar access. Tri-state as string enum not
  boolean, ≤128 fields / ≤7 nesting levels.
- **Non-throwing batch execution** (§5 Part F step 19): every AI Function
  call must capture errors, but the mechanism differs by function —
  `failOnError => false` for `ai_query`; the returned `error_message`
  VARIANT field for `ai_classify`/`ai_extract` (which don't accept
  `failOnError`). A single bad call must never abort the whole run.
- **Idempotent DDL**: `CREATE TABLE IF NOT EXISTS`, not
  `CREATE OR REPLACE`, for `rss_bronze` on redeploy (§5 Part A step 1) —
  the same caution applies to the Lakebase tables' migration mechanism
  (whatever Build Plan 00 chose must itself be redeploy-safe; this plan
  inherits that property rather than re-solving it).

## 8. Validation checklist

- [ ] `rss_bronze` is created with all five properties (CDF, Row
      Tracking, `VARIANT` column, Variant Shredding, `CLUSTER BY AUTO`)
      confirmed via `DESCRIBE TABLE EXTENDED` / `SHOW TBLPROPERTIES`, not
      just "the DDL ran without error."
- [ ] Running the polling Job twice in a row against an unchanged feed
      produces a `304` on the second run for every tracked cloud (confirms
      conditional GET is actually working, not just present in the code)
      and appends **zero** new `rss_bronze` rows on that second run.
- [ ] A feed with a genuinely new item produces exactly one new
      `rss_bronze` row per new item, and exactly one deduped `rss_silver`
      row even when the same `guid` is manually inserted into two
      different cloud feeds within the same dedup window (proves the
      cross-cloud merge, not just single-feed dedup).
- [ ] The Research Agent Job fires automatically (no manual trigger)
      within a reasonable delay after a new `rss_silver` row commits, and
      processes **exactly** the new row(s) — re-running/re-triggering the
      same Job manually a second time with no new `rss_silver` commits
      processes **zero** rows (proves the watermark, not just "it ran").
- [ ] Manually inserting two `rss_silver` rows in one Delta commit (a
      single batch write) results in the Research Agent processing
      **both** rows in one Job run (proves the CDF-based enumeration
      handles multi-row commits, not just "one trigger = one row" by
      accident).
- [ ] With the `-ai-tools` docs-corpus index absent (true for this plan's
      initial build), the Research Agent completes successfully end-to-
      end — confirms the optional-corpus no-op path (§5 Part F step 13)
      actually degrades gracefully rather than raising.
- [ ] Run the full pipeline manually against one real, current Databricks
      Release Notes RSS entry end-to-end (per `PROJECT-PLAN.md` §4's
      stated purpose for building this phase first) and manually inspect
      the resulting JSON findings file: citations resolve to real URLs
      that actually support the summary's claims, at least one
      `skill_classifications` entry has a plausible label, and the
      availability block only contains this installation's tracked
      regions.
- [ ] Force one `ai_extract` (or `ai_query`/`ai_classify`) call to fail
      (e.g. a malformed fetched page, or a temporarily unreachable URL)
      and confirm the run still completes, writes a findings file with a
      populated `errors` array for that entry, and does **not** crash the
      whole batch or skip advancing the watermark for the *other*,
      successfully-processed entries in the same run.
- [ ] No PAT, private key, or other credential appears in this Job's logs
      (the GitHub App token-minting flow from Build Plan 00 must be used
      correctly here too, not bypassed with a hardcoded token for
      convenience).

## 9. Acceptance criteria (Definition of Done)

1. The full pipeline (poll → dedup → trigger → research → JSON file) runs
   unattended on its daily schedule against at least one real tracked
   cloud feed, with no human intervention required to keep it running.
2. A genuinely new RSS entry, injected either by waiting for a real
   Databricks release-notes update or by manually inserting a
   fixture row into `rss_bronze`, results in a correct, cited,
   classified JSON findings file within one Job run cycle, with zero
   duplicate processing on subsequent unrelated trigger fires.
3. The Research Agent's behavior is correct with the `-ai-tools` Vector
   Search index absent — this plan does not implicitly depend on Build
   Plan 02 having been built first.
4. §8's validation checklist passes in full, including the multi-row-
   commit and forced-failure cases (not just the happy path).
5. The JSON findings schema (§4) and the file naming/layout convention
   (§5 Part B step 5) are both recorded as final, not left as open
   placeholders, since Build Plan 03 depends on them being stable.

## 10. Open items carried forward (not blockers, tracked for later plans)

- **Research Agent Job's bundle placement (`-infra` vs. a hypothetical
  fourth category)** — this plan made an explicit design call (§3) to
  place it in `-infra` alongside the RSS polling Job, since
  `PROJECT-PLAN.md` §1d doesn't name it directly. Revisit only if a
  future plan gives the Research Agent a genuine deploy-time dependency
  on `-ai-tools` (it currently has only an optional runtime one).
- **One shared service principal for both Jobs** (§5 Part A step 2) —
  chosen for simplicity; if least-privilege review later wants the
  polling Job's SP to *not* have GitHub-token-minting or `ai_query`/
  `ai_classify`/`ai_extract` permissions it never uses, split into two
  SPs then. Not done here to avoid over-engineering before there's a
  concrete need.
- **Whether the Research Agent SP can mint GitHub installation tokens
  itself, or must call back into the App** (§5 Part A step 3) — must be
  resolved against Build Plan 00's actual implementation before Part F
  step 15/17 are built; record the answer here once known.
- **One Job instance serving all installations vs. one Job per
  installation** (§5 Part C step 7) — must be decided and recorded before
  Part C/Part F are fully scoped; this plan's task descriptions are
  written to work either way, but the actual deployment automation
  differs meaningfully between the two models.
- **Exact JSON findings file naming/layout convention** (§5 Part B step
  5) — pick one, record it here, since Build Plan 03's Autoloader source
  path depends on it.
- **Exact daily polling Job schedule hour** (§5 Part C step 6) — pick a
  specific afternoon/evening-UTC hour and record it here once chosen.
- **`table_changes` boundary source for Part F step 11** — *resolved by
  cross-review against `databricks-jobs`*: the `table_update` trigger
  exposes no payload, so `end_version` comes from a `DESCRIBE HISTORY`-style
  version lookup at run start (Part F step 11), not a trigger-payload value;
  and the debounce fields (`min_time_between_triggers_seconds`,
  `wait_after_last_change_seconds`) do exist and are now set in Part E.
  Nothing left open here.
- **Confidence/rationale for classifications**, if the Review App (Build
  Plan 04) ends up wanting them — deliberately not built here since
  `ai_classify` doesn't provide them natively (§5 Part F step 15); would
  need a small separate LLM call added later, scoped to Build Plan 04 or
  a revision of this plan, not assumed as already present.
- **Lakebase migration mechanism** — inherited unresolved from Build Plan
  00 (§10 there); this plan's two new tables must use whatever gets
  decided there.

---

**Changelog from cross-review:** fixed a systemic off-by-10 internal
cross-reference bug where the header, §2, §3, and parts of §5 cited Part
F's steps with local 1-based numbers while §6/§7/§8 correctly used the
document's actual continuous global numbering (11–20) — standardized on
the global numbering everywhere, plus fixed a similar stray "steps 2–7"
reference inside step 11 itself; added two previously-untracked deferred
decisions to §10 (the polling schedule's exact hour, and the
`table_changes` boundary/debounce source); enumerated the previously
unsourced `no_csp` compliance-availability fetch path and de-hardcoded
the FedRAMP region list (now explicitly sourced live from the fetched
page each run, not asserted in this file); replaced the unrealistic
single-flow `dropDuplicatesWithinWatermark` + unwindowed `groupBy`
dedup/merge design with a two-chained-flow design as primary (dedup flow
+ a bounded backfill flow), since the two cannot coexist in one flow as
originally drafted; added missing CDF/Row Tracking table properties for
`rss_silver` itself (required for the CDF-based watermark read to work at
all, not inherited from `rss_bronze`); replaced the open-ended
`TABLE_CHANGES(..., start+1)` call with an explicit bounded
`[start_version + 1, end_version]` range to avoid a race where a commit
landing mid-run gets nondeterministically included/excluded; added an
explicit DBR ≥18.2 + serverless-only compute prerequisite for
`ai_query`/`ai_classify`/`ai_extract` (a stricter, separate floor from
Variant Shredding's DBR ≥17.2); pinned `ai_classify` to
`version: '2.0'` + `multilabel: 'true'` and `ai_extract` to
`version: '2.0'` explicitly (v2.1's changed per-field return shape would
otherwise silently break every `result:response:...` access in this
plan); softened the GitHub-token-minting design in Part A step 3 to
present both real options rather than implying one was already chosen;
and added the same "scaffolding parallel, grant-application dependent"
caveat from Build Plan 00's §6 to this plan's Part A/B/D fan-out
description.

**Post-review addendum:** investigated (two independent lenses) whether
`ai_prep_search` should be used in this plan's pipeline. Confirmed real
(Beta, DBR 18.2+) but its input contract is `ai_parse_document` output
only (binary document files) — doesn't fit Part F's live-fetched-HTML/
tabular inputs, so not adopted here. Noted in §5 step 16 and cross-linked
to `PROJECT-PLAN.md`'s Phase 1 note, which flags it as the right tool
*if* a future document-file-based docs corpus is added.

**Second cross-review (vs. `databricks-jobs` + `databricks-dbsql`, with a
live-docs tiebreaker):** audited this plan against the two additional
CLI-installed skills that co-own its Job/trigger and AI-Function design.
Applied from `databricks-jobs`: added the **required** `pause_status:
UNPAUSED` to the `table_update` trigger block (its omission is the skill's
#1 "trigger not firing" cause); resolved the debounce open item by setting
`min_time_between_triggers_seconds`/`wait_after_last_change_seconds`; added
`max_concurrent_runs: 1` + `queue: {enabled: true}` so a trigger firing
during an active run is queued, not silently dropped; **dropped the
unsupported "trigger payload exposes a commit version" branch** in Part F
step 11 (the skill documents a trigger payload only for `file_arrival`,
never `table_update`) and committed to the `DESCRIBE HISTORY` version
lookup; corrected the serverless-selection *mechanism* (serverless is
chosen by omitting cluster config, not by pinning a DBR); added `max_retries`
+ `email_notifications.on_failure` guidance for both Jobs; specified the
cron block's required `timezone_id`/`pause_status`; noted `for_each_task`
as the idiomatic per-installation loop; and flagged that only `ANY_UPDATED`
is documented (do not use `ALL_UPDATED` unverified). Applied from
`databricks-dbsql`: regrounded the `http_request` rejection on its
corroborated reasons (return type is only `STRUCT<status_code, text>` — no
response headers — plus the required UC HTTP `CONNECTION`/`USE CONNECTION`
privilege) and dropped the unsupported "high-volume batch" wording;
explicitly rejected `read_files` for live polling (it reads stored files,
not live HTTP); added the `ai_query` single-top-level-field `responseFormat`
constraint; added a `CLUSTER BY AUTO` recommendation for `rss_silver`; and
added a caveat against wrapping the watermark/findings sequence in a
Preview multi-statement transaction. **Live-docs tiebreaker (the key
finding):** `databricks-dbsql` and `databricks-ai-functions` disagreed on
the AI Functions API surface, so a third agent verified against live
docs.databricks.com — the richer v2 API (`ai_classify` options/version/
multilabel/500-label/array return; `ai_extract` options/version/wrapped
`result:response`, v2.1 per-field `{value, citation_ids, confidence_score}`)
and the DBR ≥ 18.2 + serverless-for-notebooks floor are all **current and
correct**, and `databricks-dbsql` is **stale** (documents the legacy v1
shapes and the older ~15.x floor). The plan was therefore **not** downgraded;
instead the divergence is recorded as a concrete instance of the exact skill
staleness this project targets. `failOnError` was corrected to `ai_query`-only
(`ai_classify`/`ai_extract` surface errors via an `error_message` VARIANT
field instead).
