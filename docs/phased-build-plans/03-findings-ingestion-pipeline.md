# Build Plan 03 — Phase 2b: Findings-Ingestion Pipeline (findings → queryable + searchable)

**Source of truth:** `docs/PROJECT-PLAN.md` §1 (architecture diagram —
Ingestion Pipeline box), §1c (Lakebase vs. Delta boundary), §1d (three-bundle
split), §2 Phase 2b, §2 Phase 6 (two-index / shared-endpoint decision), §5
(versioning frontmatter — `last_research_log_id`). This plan restates what's
needed to build Phase 2b; it does not re-litigate design decisions already
locked there — if something here seems to conflict with `PROJECT-PLAN.md`,
that file wins and this plan is stale.

**Cross-reviewed:** _pending._ A first draft; independent structural (`pi`)
and Databricks-mechanics (`codex`) passes will fold in before this is marked
ready.

**What this phase is, in one sentence:** a Lakeflow Spark Declarative
Pipeline (streaming, Autoloader) that watches the JSON findings UC Volume the
Research Agent writes to (Build Plan 01), lands each file as raw bronze, then
explodes it into the `research_log` silver Delta table (one row per skill
classification per RSS entry), after which a `research_log` Delta Sync Vector
Search index — on the **same** Storage-Optimized endpoint Build Plan 02
created — is synced so the review queue and the KB/MCP server can query it.

**Why this phase exists as its own pipeline (design rationale, not
re-litigated here):** it decouples "the Research Agent writes a file" from
"the data lands in queryable tables + a search index." The Research Agent
(Build Plan 01) stays simple and idempotent — it only ever writes a JSON file
— and all the land-it-and-index-it logic lives in this one reusable streaming
pipeline instead of being duplicated in the agent. See `PROJECT-PLAN.md` §2
Phase 2b.

> **Decisions made in this plan that `PROJECT-PLAN.md` does not pin down.**
> Phase 2b in the source doc specifies the *what* (Autoloader JSON → bronze →
> `research_log` silver → a TRIGGERED Delta Sync index that gets
> `sync_index()`'d after each batch) but leaves several *hows* open. This plan
> makes explicit, flagged choices for: the `research_log` **row-identity /
> primary-key scheme** (§4, §10 #1); **what triggers the pipeline** (§5 Part
> E, §10 #2); **where `sync_index()` actually runs** — which is *not* inside
> the LDP, for the same declarative-side-effect reason that killed the earlier
> `ForEachBatch Sink` idea (§5 Part D, §7, §10 #3); the **explode-and-upsert
> mechanism** (AUTO CDC / `apply_changes` keyed on the row id vs. plain append
> — §5 Part C, §10 #4); and **which nested fields are stored as `VARIANT`**
> vs. typed columns (§4, §10 #5). Each discretionary choice is consolidated in
> §10 for the repo owner and cross-review to confirm or override.

---

## 1. Objective

Turn the stream of JSON findings files into two durable, queryable surfaces
that every later phase depends on:

1. **`research_log`** — a Delta silver table, **one row per skill
   classification per RSS entry** (i.e. the Research Agent's
   `skill_classifications[]` array is exploded into rows), carrying the
   summary, citations, per-classification availability, proposed diff, and
   status. This is the append-oriented system of record the Review App (Build
   Plan 04) and Phase 4 read from. Per `PROJECT-PLAN.md` §1c it stays in
   Delta, **not** Lakebase.
2. **A `research_log` Delta Sync Vector Search index** — managed embeddings,
   `TRIGGERED`, on the **shared Storage-Optimized endpoint** created by Build
   Plan 02 (§2 Phase 6: the docs-corpus index and this index share one
   endpoint). Each `research_log` row is the retrieval unit; its `summary` is
   the embedded text. This backs the KB search UI and MCP server (Build Plan
   08) and, optionally, low-latency review-queue reads.

A **bronze** landing table (raw parsed JSON + ingestion metadata) sits under
`research_log` so the raw file content is always replayable and the silver
transform can be re-derived without re-reading the Volume.

Explicit non-goals (owned by other plans):

- Writing the JSON findings files — that is Build Plan 01's Research Agent.
  This plan only ever **reads** the Volume.
- Creating the JSON findings UC Volume — Build Plan 01 creates it (its Part
  B); this plan consumes it.
- Creating the shared Vector Search **endpoint** — Build Plan 02 creates it
  idempotently (it builds first); this plan creates only the **index** on it.
- The review queue / accept-reject / editor UI, and the review-queue CRUD
  state — Build Plan 04 (Phase 3). Review-queue click-state lives in Lakebase
  (§1c) and is never written back into `research_log`.

---

## 2. Prerequisites

- **Build Plan 01 is built (or its contracts are frozen).** This pipeline is
  the downstream consumer of Build Plan 01's **JSON findings file schema**
  (its §4) and **file naming/layout convention** (its Part B step 5). Those
  two are a hard contract — see §3 below, which reproduces the schema as this
  plan understands it. If Build Plan 01's schema changed, this plan is stale.
- **Build Plan 02 is built (or its shared-endpoint contract is frozen).** This
  plan's index is created on the **shared Storage-Optimized Vector Search
  endpoint** and must use the **same pinned managed-embedding endpoint** Build
  Plan 02 pinned. Both are shared config, not re-decided here (§6).
- **The `-infra` bundle exists** (UC catalog/schema; the JSON findings UC
  Volume from Build Plan 01; the shared infra-jobs service principal
  introduced in Build Plan 01 §3). This plan adds the bronze + `research_log`
  tables and the ingestion pipeline to `-infra`.
- **The `-ai-tools` bundle exists** (Build Plan 02 stood it up with the shared
  endpoint + docs-corpus index). This plan adds the `research_log` index to
  it.
- **DBR / runtime floor:** the pipeline runs on serverless Lakeflow Declarative
  Pipeline compute. If any `VARIANT` columns use variant shredding (§4, §10
  #5), the DBR ≥ 17.2 + workspace Previews-toggle floor from `PROJECT-PLAN.md`
  §2 Phase 2 applies — confirm the pipeline's runtime satisfies it, or drop
  shredding for the first cut.
- **Build order:** per `PROJECT-PLAN.md` §4, Phase 2b is built after Phase 2
  (Build Plan 01) and Phase 1 (Build Plan 02). It has no dependents that can't
  wait: Build Plan 04 (Review App) needs `research_log` to have a handful of
  real rows before it's worth building.

---

## 3. The input contract (reproduced from Build Plan 01 §4 — do not diverge)

This pipeline's Autoloader source reads files written by Build Plan 01's
Research Agent. **This is that plan's authoritative schema; if the two ever
disagree, Build Plan 01 §4 wins.** Reproduced here so an implementer building
Phase 2b in isolation has the exact shape:

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

Contract facts this pipeline must honor (all inherited from Build Plan 01 §4):

- **One file per RSS entry**, named/laid out per Build Plan 01 Part B step 5:
  `/Volumes/main/<schema>/research_findings/<installation_id>/<rss_guid_hash>.json`
  (confirm the exact convention against Build Plan 01's §10 once it's frozen).
  Autoloader globs the whole `research_findings/**` tree.
- **`skill_classifications` is an array** — the silver transform explodes it,
  so one file becomes N `research_log` rows (N ≥ 1; a "New Feature" entry may
  have exactly one classification with `target_skill_path: null`).
- **Compliance is a string tri-state (`"true"`/`"false"`/`"unknown"`), not a
  boolean.** Preserve it as strings end-to-end — do not coerce to boolean.
  Build Plan 01 deliberately chose the enum so Autoloader doesn't infer an
  inconsistent column type across files. Compliance is keyed by **dynamic
  region names** (only this installation's tracked regions appear) — that
  dynamic-key shape is the main reason `availability` is a `VARIANT`/map, not
  a fixed struct (§4, §10 #5).
- **`schema_version`** exists so this pipeline can branch on shape later; the
  bronze table must keep it as a first-class column and the pipeline should
  assert/route on it rather than silently assuming `"1"`.
- **`target_skill_path` is `null`** for a "New Feature" or user-created bucket
  with no existing skill file; `proposed_diff` may still carry drafted new-file
  content in that case. Do not drop such rows.
- **`errors` is a possibly-non-empty array** even on an otherwise-successful
  entry (partial per-step failure). Carry it through to `research_log` intact;
  do not treat a populated `errors` as a reason to reject the row.

---

## 4. Data model

### Delta / Unity Catalog (new in this plan, both in `-infra`)

**`research_findings_bronze`** — raw landing, append-only, one row per
ingested file. Autoloader target.

| Column | Type | Notes |
|---|---|---|
| `_source_file` | `STRING` | `_metadata.file_path` — the Volume path of the file this row came from. Natural dedup/debug key. |
| `_ingested_at` | `TIMESTAMP` | Autoloader ingest time. |
| `schema_version` | `STRING` | Lifted to top level for cheap routing/asserts. |
| `payload` | `VARIANT` | The full parsed JSON object (see §10 #5 — `VARIANT` vs. an inferred struct). |
| `_rescued_data` | `STRING` | Autoloader rescued-data column for anything off-schema; a non-null value here is an observability signal (Build Plan 07), not a silent drop. |

**`research_log`** — silver, **one row per (RSS entry × classification
label)**. System of record for Build Plan 04 / Phase 4. Stays in Delta (§1c).

| Column | Type | Notes |
|---|---|---|
| `research_log_id` | `STRING` (PK) | **Deterministic, content-addressed** (§10 #1): `rl_` + a stable short hash of (`installation_id`, `rss_guid`, `label`). Stable across re-ingestion → idempotent upsert + a stable Vector Search primary key. See §10 #1 for the reconciliation with §5's `last_research_log_id` display format. |
| `installation_id` | `STRING` | From the file. |
| `research_agent_run_id` | `STRING` | From the file. |
| `rss_guid` | `STRING` | `rss_entry.guid`. |
| `rss_link` | `STRING` | `rss_entry.link`. |
| `rss_title` | `STRING` | `rss_entry.title`. |
| `pub_date` | `TIMESTAMP` | `rss_entry.pub_date`. |
| `source_feed_clouds` | `ARRAY<STRING>` | Which feed(s) surfaced this entry. |
| `processed_at` | `TIMESTAMP` | When the Research Agent processed it. |
| `summary` | `STRING` | Markdown, inline `[n]` citations. **This is the embedding source column** for the index. |
| `label` | `STRING` | The exploded classification label (a skill name, `"New Feature"`, or a user bucket). Filterable index metadata. |
| `is_new_feature_bucket` | `BOOLEAN` | From the classification. |
| `target_skill_path` | `STRING` (nullable) | `null` for New Feature / new bucket. |
| `proposed_diff` | `STRING` (nullable) | Unified diff, or drafted new-file content, or null. |
| `citations` | `VARIANT` | Array of `{id, url, title}`; read by the review card's footnotes. |
| `availability` | `VARIANT` | `{clouds[], compliance{...}}` with dynamic region keys — `VARIANT`/map, not a fixed struct (§10 #5). Tri-state stays string. |
| `errors` | `VARIANT` | Carried through from the file (may be empty array). |
| `status` | `STRING` | From the file (`"pending_review"` at ingest). **Read-only lineage of the finding's origin — NOT the review decision.** The accept/reject/edit click-state is Lakebase-owned (§1c) and never written back here. |
| `schema_version` | `STRING` | Propagated from bronze. |
| `_ingested_at` | `TIMESTAMP` | For observability / late-arrival debugging. |

Table properties on `research_log` (declared in the pipeline — modern SDP
tables are full Delta tables and accept these; confirm at build against the
pipeline's runtime):

- `delta.enableChangeDataFeed = true` — CDF is the documented Delta Sync
  requirement for standard endpoints; for the Storage-Optimized endpoint used
  here it may not be a hard gate, but enable it regardless (harmless, enables
  incremental sync). Confirm the current Storage-Optimized requirement against
  official docs, per Build Plan 02 §7.
- `delta.enableRowTracking = true` and `CLUSTER BY AUTO` — consistent with the
  project's table conventions (§ `rss_bronze` in `PROJECT-PLAN.md` §2). If any
  `VARIANT` column uses variant shredding, add `delta.enableVariantShredding`
  and satisfy the DBR ≥ 17.2 floor (§10 #5).

### Vector Search — `research_log` index (`-ai-tools`)

- **Delta Sync index, managed embeddings, `TRIGGERED`**, on the **shared
  Storage-Optimized endpoint** (Build Plan 02). Do **not** create a second
  endpoint.
- `primary_key = research_log_id`.
- **Embedding source column = `summary`** (the retrieval unit is the whole
  finding's summary; `ai_prep_search`-style chunking is deliberately **not**
  used here — a `research_log` row is already the atomic retrieval unit, per
  Build Plan 01's note at its Part F step 16).
- `columns_to_sync`: lightweight **filterable scalars only** — `label`,
  `installation_id`, `status`, `is_new_feature_bucket`, `pub_date`,
  `target_skill_path`, `rss_link`. The rich `VARIANT` columns (`citations`,
  `availability`, `errors`) are **not** synced into the index — the Review App
  and MCP server read those from the `research_log` Delta row by
  `research_log_id`. (Confirm whether `VARIANT` metadata columns are even
  syncable; scoping the index to scalars sidesteps the question — §7.)
- Pinned managed-embedding endpoint = **the same one Build Plan 02 pinned**
  (shared config, §6). Resolve from official docs, not the stale local
  `databricks-vector-search` skill.

### Lakebase

**None new in this plan.** `research_log` stays in Delta (§1c). The only
Lakebase interaction Phase 2b touches indirectly is that Build Plan 04's
review-queue state will reference `research_log_id` as a foreign key — which
is exactly why §10 #1 (a *stable* `research_log_id`) matters across plans.

---

## 5. Task breakdown

### Part A — `research_findings_bronze` + `research_log` tables (`-infra`)

1. Declare the bronze and silver tables (§4) as pipeline datasets in the
   findings-ingestion LDP's source (Python `pyspark.pipelines` — modern SDP,
   **not** legacy `import dlt`/`LIVE.`). Set `research_log`'s table properties
   (CDF, row tracking, `CLUSTER BY AUTO`, optional variant shredding) in the
   table definition. Grants: the shared infra-jobs SP (Build Plan 01 §3) owns
   and runs the pipeline; Build Plan 04's App SP gets `SELECT` on
   `research_log` (via the Analytics path or a synced-table GRANT, decided in
   Build Plan 04).

### Part B — Bronze flow: Autoloader JSON → `research_findings_bronze` (`-infra`)

2. **Autoloader (`cloudFiles`, format `json`) streaming read** of the findings
   Volume glob (`/Volumes/.../research_findings/**`, exact path per Build Plan
   01's frozen convention). Configure:
   - A **schema location** for Autoloader's schema tracking, and
     **`schemaHints`** pinning the known top-level shape (esp. the compliance
     tri-state as `STRING`, and `skill_classifications` as an array) so
     inference can't flip a column type across files where some entries are
     `"unknown"` and others `"true"`/`"false"`.
   - **`_rescued_data`** enabled — off-schema content is captured, not dropped.
   - **`cloudFiles.schemaEvolutionMode`** chosen deliberately (default
     `addNewColumns` vs. `rescue`) — record the choice; a schema change should
     surface as a rescued column / observability signal (Build Plan 07), not a
     crash.
3. Write raw rows to `research_findings_bronze` with `_source_file`
   (`_metadata.file_path`), `_ingested_at`, `schema_version` lifted to top
   level, and the full object in `payload` (`VARIANT` — see §10 #5).
4. **Idempotency of file ingestion:** Autoloader's checkpoint tracks
   already-seen files exactly-once. Decide and record behavior for a
   **re-written file** (same `<rss_guid_hash>.json` overwritten by a
   corrected re-run): default Autoloader may **not** re-ingest an overwritten
   path. If corrections must flow, either set `cloudFiles.allowOverwrites` or
   rely on the deterministic `research_log_id` upsert (Part C) to make a
   re-ingested file converge rather than duplicate. Cross-check against Build
   Plan 01's watermark design, which normally prevents re-processing the same
   `rss_silver` row at all (§10 #4).

### Part C — Silver flow: explode → `research_log` (`-infra`)

5. **Streaming transform** bronze → silver:
   - `explode(skill_classifications)` so one file → N rows.
   - Project each row's columns per §4: lift `rss_entry.*`, `summary`,
     `citations`, top-level fields, and the per-classification `label`,
     `is_new_feature_bucket`, `target_skill_path`, `availability`,
     `proposed_diff`.
   - Compute `research_log_id = 'rl_' || sha2(concat_ws(':', installation_id,
     rss_guid, label), 256)` (short-hashed — §10 #1). Deterministic → the same
     finding re-ingested produces the same id.
   - Keep compliance strings **as strings**; keep `citations`/`availability`/
     `errors` as `VARIANT` (§10 #5).
6. **Upsert semantics (primary design — §10 #4):** land into `research_log`
   via LDP **AUTO CDC / `apply_changes`** (SCD type 1) keyed on
   `research_log_id`, so a re-ingested or corrected finding **updates in place**
   rather than duplicating. Sequence by `processed_at` (or `_ingested_at`) so
   the latest wins. Alternative if AUTO CDC proves unnecessary: a plain
   append streaming table, relying on Build Plan 01's watermark to guarantee
   one-file-per-entry and thus no dupes — recorded as the fallback in §10 #4.
7. **`sync_index()` is NOT called from inside this pipeline.** An LDP dataset
   function returns a DataFrame; it has no sanctioned way to perform the
   external Vector Search API side-effect mid-flow — this is the *same*
   declarative-side-effect constraint that led `PROJECT-PLAN.md` §2 Phase 2b
   to **reject** the `ForEachBatch Sink` idea. So the sync is an **orchestration
   step after the pipeline update lands**, not a pipeline step. See Part E.

### Part D — `research_log` Vector Search index (`-ai-tools`)

8. Declare the `research_log` Delta Sync index (§4) as a **DAB resource**
   (`resources.vector_search_indexes`, confirmed declarable in the CLI v1.5.0
   bundle schema per Build Plan 02 §3) on the shared endpoint, with
   `primary_key = research_log_id`, embedding source `summary`, `TRIGGERED`
   pipeline type, and the pinned managed-embedding endpoint (§6). Follow the
   **same declared-resource approach** Build Plan 02 used for the docs-corpus
   index; the two can share a common helper (Build Plan 02 §3 anticipates
   this).
9. Do **not** create the endpoint here — reference the shared one Build Plan
   02 created. An idempotent post-deploy setup job is a contingency only (as
   in Build Plan 02), not the default.

### Part E — Orchestration: run the pipeline, then sync the index (`-infra` job + `-ai-tools` index)

10. **What triggers the pipeline (§10 #2).** Primary proposal: make the
    findings-ingestion pipeline a **downstream task of the Research Agent Job**
    (Build Plan 01) — the agent writes all findings files (its Part F step
    20), then a dependent task runs this pipeline (`availableNow`-style
    one-shot streaming update), then a final task calls `sync_index()`. This
    reuses the Research Agent's existing trigger (the `rss_silver`
    `table_update` trigger) end-to-end, and guarantees the sync runs *after*
    the batch lands. Alternative: a `file_arrival` trigger on the findings
    Volume path (Volumes are a documented `file_arrival` scope, unlike managed
    tables) driving a standalone ingestion Job. Record the choice; cross-review
    should confirm which is more robust against partial-batch runs.
11. **Call `sync_index()` after the pipeline update** — a Job task (Python,
    Databricks SDK / Vector Search client) that invokes the current sync
    operation on the `research_log` index (confirm the current API name given
    the Vector Search → AI Search rename, per Build Plan 02 §5 Part E). This
    is the "TRIGGERED needs an explicit sync" step from `PROJECT-PLAN.md` §2
    Phase 2b — placed here, not in the LDP (Part C step 7). Make it idempotent
    and non-fatal-on-no-op (a run with zero new rows should sync cleanly or
    skip, not error).

---

## 6. Cross-plan contracts (shared config — do not re-decide here)

- **JSON findings schema + file-layout convention (Build Plan 01 §4 / Part B
  step 5).** This plan's Autoloader source and bronze schema are downstream of
  it. If Build Plan 01 changes either, this plan must change in lockstep;
  `schema_version` is the agreed lever for evolving it without a hard break
  (§3).
- **Shared Storage-Optimized Vector Search endpoint (Build Plan 02 §6).** One
  endpoint, created by Build Plan 02 (it builds first); this plan creates only
  the `research_log` index on it. Neither plan hardcodes a second endpoint.
- **Pinned managed-embedding endpoint (Build Plan 02 §6).** Both indexes use
  the same resolved endpoint, pinned once in shared config, so they don't
  silently diverge.
- **Consumer: Build Plan 04 (Phase 3, Review App).** Reads `research_log`
  (through the Apps Data Access Decision Gate — warehouse Analytics by
  default, optional Lakebase synced-table mirror if too slow, per
  `PROJECT-PLAN.md` §2 Phase 3). Build Plan 04's review-queue CRUD state keys
  on `research_log_id` — so **`research_log_id` must be stable** (§10 #1). The
  interface is just "the `research_log` schema + a stable id" — no code
  coupling.
- **Consumer: Build Plan 08 (Phase 6, stretch).** The KB search UI / MCP
  server queries the `research_log` index; default queries to **HYBRID**
  (`PROJECT-PLAN.md` §2 Phase 6). Nothing to build for it here beyond keeping
  `columns_to_sync` rich enough to support attributed results (`label`,
  `rss_link`, `pub_date` are included for that reason).
- **Producer contract with Build Plan 01's watermark.** Build Plan 01 advances
  its `rss_silver_watermark` only after all findings files for a run are
  written (its Part F step 20). This plan should be able to assume a run's
  files are complete when it fires — relevant to the trigger choice (§5 Part
  E / §10 #2).

---

## 7. Platform constraints to build against (to be cross-review-confirmed)

- **LDP cannot perform the Vector Search side-effect.** A declarative dataset
  function returns a DataFrame; it has no sanctioned external-write hook, and
  the one streaming external-write mechanism (a `Sink`) is append-only, not an
  index API call. This is why `sync_index()` is an orchestration step, not a
  pipeline step (§5 Part C step 7 / Part E) — and why the earlier
  `ForEachBatch Sink` design was rejected (`PROJECT-PLAN.md` §2 Phase 2b).
- **`TRIGGERED` Delta Sync index requires an explicit sync call** after each
  batch — it does not sync on every table change automatically. Confirm the
  current API/operation name (Vector Search → AI Search rename).
- **CDF requirement is endpoint-tier-dependent.** CDF is the documented Delta
  Sync requirement for **standard** endpoints; for the **Storage-Optimized**
  endpoint used here it may not be a hard gate — enable it on `research_log`
  regardless (harmless + incremental sync), but confirm the current
  Storage-Optimized requirement against official docs, not the stale local
  skill (same finding as Build Plan 02 §7).
- **Autoloader schema inference across mixed files** can flip a column's type
  when some files carry `"unknown"` and others `"true"`/`"false"` — pinned
  `schemaHints` + the string-enum contract (§3) prevent this. Keep a schema
  location so inference is stable across restarts.
- **`explode` in a streaming flow** is a stateless transform — valid in a
  streaming table. AUTO CDC / `apply_changes` (§5 Part C step 6) requires the
  target to be a streaming table declared in the pipeline; confirm the modern
  `pyspark.pipelines` API surface for it (not the legacy `dlt.apply_changes`).
- **`VARIANT` metadata columns in a Vector Search index** may not be syncable —
  scoping `columns_to_sync` to scalars (§4) sidesteps this; confirm before
  adding any `VARIANT` to the index.
- **Modern SDP-created tables are full Delta tables** and accept CDF / row
  tracking / `CLUSTER BY AUTO` / variant-shredding TBLPROPERTIES (locked
  decision, `PROJECT-PLAN.md` — "review only OSS Spark 4+ SDP, never legacy
  DLT"). Set them in the table definition; confirm the pipeline runtime
  satisfies any preview floor (variant shredding → DBR ≥ 17.2 + Previews
  toggle).

---

## 8. Explicit fan-out map

**Within this plan:**

- **Track 1 — ingestion tables + pipeline (`-infra`)**: Part A → Part B →
  Part C are a single dependent chain (bronze schema → silver transform); one
  worker builds them together since the silver transform is written against
  the bronze shape.
- **Track 2 — index (`-ai-tools`)**: Part D (declare the `research_log` index)
  depends only on the **shape** of `research_log` (its PK + `summary` column),
  not on the pipeline being runnable, so it can be authored in parallel with
  Track 1 once §4 is fixed — but it can't be *deployed/validated* until Track
  1 has produced a `research_log` table with at least one row.
- **Track 3 — orchestration (`-infra` job + sync task)**: Part E depends on
  both tracks existing (it runs the pipeline, then syncs the index). Author
  last; it's thin.
- Tracks 1 and 2 are the natural two-worker parallel split; Track 3 joins them.

**Across plans:**

- **Upstream, must be frozen first:** Build Plan 01 §4 (JSON schema) + Part B
  step 5 (file layout), and Build Plan 02 §6 (shared endpoint + embedding
  pin). This plan cannot be *validated* end-to-end until Build Plan 01 can
  actually drop a real findings file and Build Plan 02's endpoint exists — but
  it can be *authored* against the frozen contracts in parallel with late work
  on 01/02.
- **Downstream consumers:** Build Plan 04 (Review App) and Build Plan 08
  (KB/MCP) both read `research_log` / its index. Neither couples in code —
  the interface is the `research_log` schema + a stable `research_log_id` +
  the index name.

---

## 9. Validation checklist

Drive the pipeline end-to-end with a **known, hand-crafted findings file**,
not just unit tests:

- [ ] Drop a findings file with **two** `skill_classifications` into the
      Volume → assert exactly **two** `research_log` rows, each with the
      correct exploded `label` / `target_skill_path` / `availability`, and
      identical shared fields (`summary`, `citations`, `rss_*`).
- [ ] **Compliance tri-state survives as strings** — a `"unknown"` in the
      source file is `"unknown"` in `research_log`, not `null`/`false`/a
      boolean, and does not corrupt the column type for other rows.
- [ ] **`research_log_id` is deterministic** — re-ingest the same file (force
      re-read) and assert the row **upserts** (same id, no duplicate row); the
      row count is unchanged.
- [ ] A file with a `null` `target_skill_path` (New Feature bucket) and a
      populated `proposed_diff` **is not dropped** — it lands as a row.
- [ ] A file with a **non-empty `errors` array** lands normally (errors carried
      through, row not rejected).
- [ ] A **malformed / off-schema** file surfaces in `_rescued_data` (or the
      chosen evolution mode's signal) and does **not** crash the pipeline; the
      other files in the batch still land.
- [ ] After the ingestion run, the `sync_index()` orchestration step runs and
      the **new row is queryable** in the `research_log` index (a HYBRID query
      on a phrase from `summary` returns it, with `label`/`rss_link` metadata).
- [ ] The `research_log` index is created on the **shared** endpoint (not a
      second one) and uses the **same pinned embedding endpoint** as the
      docs-corpus index.
- [ ] A run with **zero new files** completes cleanly — pipeline no-ops and the
      sync step is a safe no-op, neither errors.

---

## 10. Open items carried forward (decisions to confirm in cross-review)

1. **`research_log_id` scheme (primary design: deterministic content-hash).**
   `rl_` + `sha2(installation_id:rss_guid:label)` gives stable idempotent
   upsert + a stable Vector Search PK + a stable FK for Build Plan 04. But
   `PROJECT-PLAN.md` §5's frontmatter example shows `last_research_log_id:
   "rl_00042"` — a **sequential** display format. Reconcile: either (a) the
   frontmatter simply stores whatever id scheme this plan picks (drop the
   implied sequence), or (b) add a separate human-facing sequence column for
   display while keeping the hash as the real key. Flagged because the id is a
   cross-plan contract (Build Plan 04 / Phase 4 write `last_research_log_id`).
2. **Pipeline trigger mechanism.** Primary: a downstream task of the Research
   Agent Job (reuses its `rss_silver` `table_update` trigger, guarantees sync
   runs after the batch). Alternative: a `file_arrival` trigger on the findings
   Volume driving a standalone Job. Confirm which is more robust against
   partial-batch runs.
3. **Where `sync_index()` runs.** Decided: an **orchestration Job task after**
   the pipeline update, **not** inside the LDP (the declarative-side-effect
   constraint that killed `ForEachBatch Sink`). Confirm this is the idiomatic
   pattern and that there's no supported post-flow hook that would be cleaner.
4. **Explode-and-upsert mechanism.** Primary: AUTO CDC / `apply_changes` (SCD
   1) keyed on `research_log_id` for idempotent in-place updates. Fallback: a
   plain append streaming table relying on Build Plan 01's watermark to
   guarantee one-file-per-entry. Confirm AUTO CDC is worth the complexity vs.
   append given the watermark already prevents most re-processing.
5. **`VARIANT` vs. typed columns for `payload` / `citations` / `availability`
   / `errors`.** Primary: `VARIANT` — consistent with the repo owner's stated
   preference and the `rss_bronze` precedent, and genuinely better than a
   fixed struct for `availability` (dynamic region keys). Confirm: (a) the
   pipeline runtime satisfies the variant-shredding preview floor if shredding
   is enabled (DBR ≥ 17.2 + Previews toggle), and (b) none of these `VARIANT`
   columns need to be a Vector Search index metadata column (they aren't in
   `columns_to_sync` — §4).
6. **Autoloader re-write behavior** (`cloudFiles.allowOverwrites` vs. relying
   on the deterministic-id upsert) — see §5 Part B step 4; couple the decision
   to #4.

---

## 11. Changelog

- **Initial draft** (pre-cross-review): drafted from `PROJECT-PLAN.md` §2
  Phase 2b, §1c, §1d, §5, and the frozen contracts in Build Plan 01 §4
  (JSON findings schema) and Build Plan 02 §6 (shared endpoint + embedding
  pin). Made explicit flagged decisions for the six open items in §10 that the
  source doc leaves open — most notably placing `sync_index()` in
  orchestration rather than inside the LDP (the same declarative-side-effect
  reason the `ForEachBatch Sink` was rejected), a deterministic content-hash
  `research_log_id`, and AUTO CDC upsert on that id.
