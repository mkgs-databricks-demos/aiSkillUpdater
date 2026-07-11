# Build Plan 02 — Phase 1: Grounding Corpus (docs ingestion)

**Source of truth:** `docs/PROJECT-PLAN.md` §1 (architecture diagram),
§1c, §1d, §2 Phase 1, §2 Phase 6 (two-index / shared-endpoint decision),
§3, §5, §6. This plan restates what's needed to build Phase 1; it does not
re-litigate design decisions already locked there — if something here
seems to conflict with `PROJECT-PLAN.md`, that file wins and this plan is
stale.

**Cross-reviewed:** _pending_ — an independent structural pass (`pi`) and
an independent Databricks-mechanics fact-check pass (`codex`) will review a
first draft of this plan; every BLOCKING finding from both will be folded
into the version below. See the changelog at the bottom of this file.

**Why this phase is built second (per `PROJECT-PLAN.md` §4's build
order):** the suggested order runs Phase 2 (Build Plan 01) *before* Phase 1
because the Research Agent needs to prove its fetch → summarize → cite →
classify → propose loop against one real RSS entry before a complete docs
corpus exists. Consequently, **this corpus is supplementary grounding, not
the primary source** for any announcement — the Research Agent always
live-fetches an announcement's own links first (Build Plan 01 §5 Part F),
and only queries this corpus for *broader surrounding context*. That has a
hard consequence this plan is built around: **Build Plan 01 already treats
the corpus lookup as optional and best-effort** (its Part F step 13), so
this plan must never be a hard runtime dependency of Phase 2 — an empty,
stale, or absent corpus must degrade Phase 2 gracefully, not break it (see
§5 Part F).

> **Decisions made in this plan that `PROJECT-PLAN.md` does not pin down.**
> Phase 1 in the source doc specifies the *what* (scrape docs into a UC
> Volume, chunk + embed into a Delta Sync index with managed embeddings)
> but leaves the *how* open. This plan makes explicit, flagged choices for:
> the scrape mechanism (a Job, not a pipeline — §5 Part C), the chunking
> mechanism (Job + `MERGE`, with an Autoloader-LDP alternative — §5 Part
> D), scrape-state storage (a Delta companion table, not Lakebase — §4 and
> §10 #3), the URL/section scope (a bounded, config-driven seed set — §10
> #1), and the refresh cadence (weekly — §10 #2). Each is called out where
> it appears and consolidated in §10 for the repo owner and cross-review to
> confirm or override.

---

## 1. Objective

Stand up the supplementary docs-grounding corpus and its semantic-search
index, so the Research Agent (Build Plan 01) has broader
`docs.databricks.com` context available when it reasons about an
announcement:

1. A scheduled Job scrapes a bounded, configurable set of
   `docs.databricks.com` (and Azure `learn.microsoft.com`) pages —
   scoped to this installation's `tracked_clouds` (Build Plan 00) — using
   per-URL conditional GET, and writes cleaned page text as files into a
   UC Volume.
2. A chunking step turns those pages into heading-aware text chunks and
   `MERGE`s them (idempotently) into a `docs_corpus` Delta table (Change
   Data Feed enabled).
3. A **Delta Sync Vector Search index with managed embeddings**
   (`TRIGGERED`) sits on `docs_corpus`, on a **shared Storage-Optimized
   Vector Search endpoint** that Build Plan 03's `research_log` index will
   also use.
4. The index name is published as a config value that Build Plan 01's
   Research Agent reads — with an explicit contract that a missing/empty
   index is a no-op for Phase 2, not an error.

Out of scope for this plan: the `research_log` index and findings-ingestion
pipeline (Build Plan 03, Phase 2b — a *separate* index on a *separate*
source table, deliberately, per `PROJECT-PLAN.md` §2 Phase 6); the Research
Agent itself (Build Plan 01); and the App's Knowledge-Base search UI /
MCP server (Build Plan 08, Phase 6), which is a later *consumer* of this
same index, not built here.

## 2. Prerequisites

- **Build Plan 00 (Phase 0) deployed at least once.** This plan reads two
  things it produces:
  - `tracked_clouds` (Lakebase) — which of AWS / GCP / Azure docs domains
    to scrape (mirrors the RSS feed-selection design; the corpus should
    not scrape clouds the installation doesn't track).
  - The `-infra` and `-ai-tools` bundle roots, catalog/schema, and the
    shared infra-jobs service principal (see §3).
- **`-infra` bundle deployed and run at least once**, per
  `PROJECT-PLAN.md` §1d's strict deploy order (`-infra` → `-ai-tools` →
  `-app`). The `docs_corpus` Delta table (this plan's Part B) must exist
  and ideally have been populated once (Part D run) before the Delta Sync
  index in `-ai-tools` (Part E) is created against it.
- **A managed-embedding Foundation Model endpoint available in the
  workspace.** Resolve the *current* recommended managed-embedding
  endpoint from the `databricks-vector-search` skill at build time — do
  **not** hardcode a legacy GTE/BGE name. `databricks-qwen3-embedding-0-6b`
  is current as of this writing (`PROJECT-PLAN.md` §2 Phase 1); confirm
  before building.
- **Serverless compute** for the scrape/chunk Job (same profile as Build
  Plan 01's RSS Job — an outbound HTTP fetch plus a Spark/Delta write).
- **This plan and Build Plan 03 must agree on the shared Vector Search
  endpoint** (name, Storage-Optimized tier) and on the pinned embedding
  endpoint. See §6 for the cross-plan coordination contract.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

This plan, like Phase 1 in the source doc, spans **two** of the three
bundles:

| Resource | Bundle | Rationale |
|----------|--------|-----------|
| docs-corpus UC Volume (raw scraped pages) | `-infra` | Foundation storage; nothing data-dependent |
| `docs_corpus` Delta table + `docs_scrape_state` table | `-infra` | Source tables the index later syncs from |
| Docs-scrape Job (fetch → clean → write files) | `-infra` | Only writes `-infra`-owned Volume/tables |
| Chunk step (files → `docs_corpus` `MERGE`) | `-infra` | Produces the index's source table |
| Shared Storage-Optimized Vector Search **endpoint** | `-ai-tools` | Serving infra; also used by Build Plan 03 |
| `docs_corpus` Delta Sync **index** | `-ai-tools` | Requires the source table to exist (and ideally have rows) first |
| Post-deploy index setup job (fallback if index isn't a DAB resource) | `-ai-tools` | Idempotent create/update of endpoint + index |

- **UC Volume access uses `grants`, not a bundle `permissions` block**
  (`PROJECT-PLAN.md` §2 Phase 1, cross-reviewed vs. `databricks-dabs`):
  the infra-jobs SP needs `READ_VOLUME` + write privileges on the Volume;
  the App / Research Agent SP needs at least read.
- **The Vector Search index may not be a declarable DAB resource type**
  (`PROJECT-PLAN.md` §2 Phase 1). Verify against the current
  `databricks bundle schema`. If unsupported, deploy the endpoint/index via
  an **idempotent post-deploy setup job** (safe to re-run on every deploy),
  exactly as Build Plan 03 will for the `research_log` index — and the two
  should share that job or a common helper rather than duplicating it.
- **Service principal:** reuse the shared infra-jobs SP created in Build
  Plan 01 (§1d creates a dedicated SP for the RSS Job; this corpus Job has
  the same "writes only `-infra` tables/volumes" profile). Do **not** mint
  a new one unless cross-review finds a permissions reason to isolate them.

## 4. Data model

### Delta / Unity Catalog (new in this plan)

**`<catalog>.<schema>.docs_corpus`** — the chunked corpus; the Delta Sync
index's source table. One row per chunk.

| Column | Type | Notes |
|--------|------|-------|
| `chunk_id` | `STRING` | **Primary key** for the Delta Sync index. Deterministic: `sha2(url ‖ '#' ‖ chunk_ordinal, 256)` so re-chunking a changed page overwrites the same ids rather than accumulating orphans (see §5 Part D for the orphan-deletion caveat) |
| `url` | `STRING` | Canonical source URL |
| `source_cloud` | `STRING` | `aws` / `gcp` / `azure` — which docs domain it came from (so retrieval can filter to relevant clouds) |
| `doc_title` | `STRING` | Page title |
| `section_path` | `STRING` | Heading breadcrumb (e.g. `Jobs > Triggers > Table update`) — preserved so a retrieved chunk carries its location |
| `chunk_ordinal` | `INT` | 0-based position of the chunk within the page |
| `chunk_text` | `STRING` | The embedded/retrieved text (managed embeddings compute the vector from this column) |
| `content_hash` | `STRING` | Hash of the *source page's* cleaned text, copied onto every chunk — lets the chunk step skip unchanged pages |
| `fetched_at` | `TIMESTAMP` | When the source page was last fetched |

- **`TBLPROPERTIES`**: `delta.enableChangeDataFeed = true` (**required** —
  a Delta Sync index needs CDF on its source table; confirm exact
  requirement against `databricks-vector-search`). `CLUSTER BY AUTO` for
  clustering (`PROJECT-PLAN.md` conventions). Row Tracking optional; CDF is
  the hard requirement here.
- **Chunk sizing must respect the embedding endpoint's max input tokens.**
  Size chunks (e.g. by heading section, then a token-bounded splitter) so
  no `chunk_text` exceeds the pinned managed-embedding model's limit —
  otherwise the sync truncates or errors. Confirm the current limit for the
  resolved endpoint at build time.

**`<catalog>.<schema>.docs_scrape_state`** — per-URL scrape cursor
(conditional-GET + change detection). One row per tracked URL.

| Column | Type | Notes |
|--------|------|-------|
| `url` | `STRING` | Primary key |
| `source_cloud` | `STRING` | `aws` / `gcp` / `azure` |
| `etag` | `STRING` | From the last successful fetch, for `If-None-Match` |
| `last_modified` | `STRING` | From the last successful fetch, for `If-Modified-Since` |
| `content_hash` | `STRING` | Cleaned-text hash of the last fetched version (drives "did this page actually change") |
| `http_status` | `INT` | Last response status (200 / 304 / 4xx / 5xx) |
| `last_fetched_at` | `TIMESTAMP` | Last fetch attempt |
| `last_error` | `STRING` | Nullable; last fetch/parse error for observability (Phase 7) |

> **Decision — scrape state lives in Delta, not Lakebase.** `PROJECT-PLAN.md`
> §1c routes the *RSS* polling cursor to Lakebase because it is tiny (1–3
> rows, one per feed) and App-adjacent. The docs-scrape cursor is a
> different profile: potentially **thousands** of rows (one per scraped
> URL), written only by an `-infra` batch Job, never read interactively by
> the App. Colocating it in Delta next to `docs_corpus` (same Job, same
> bundle, same lifecycle) is the better fit and keeps Lakebase for genuine
> OLTP/App state. Flagged in §10 #3 for cross-review to confirm against
> §1c's intent.

### Lakebase

**None new in this plan.** (See the decision above — the docs-scrape cursor
intentionally stays in Delta.)

## 5. Task breakdown

### Part A — docs-corpus UC Volume (`-infra`)

1. Create a UC **managed Volume** (e.g. `<catalog>.<schema>.docs_corpus_raw`)
   to hold cleaned page files written by the scrape Job.
2. Grant the infra-jobs SP `READ_VOLUME` + write on the Volume via bundle
   `grants` (not `permissions`). Grant the Research Agent / App SP read if
   any downstream step needs the raw files (the index reads `docs_corpus`,
   not the Volume, so read-only is likely sufficient / optional).

### Part B — `docs_corpus` + `docs_scrape_state` tables (`-infra`)

3. Create `docs_corpus` with the schema in §4, including
   `delta.enableChangeDataFeed = true` and `CLUSTER BY AUTO`.
4. Create `docs_scrape_state` with the schema in §4.
5. Grant the infra-jobs SP read/write on both; grant the Research Agent /
   App SP read on `docs_corpus` if needed for debugging (the index is the
   normal read path).

### Part C — Docs-scrape Job (`-infra`)

The scrape is a **plain Job, not a pipeline** — Lakeflow Declarative
Pipelines have no HTTP source, and `http_request` inside an LDP was already
rejected for this project (`PROJECT-PLAN.md` — the RSS/LDP investigation:
no response headers exposed, so conditional GET is impossible from it, plus
LDP's retry/refresh model re-fires side-effecting fetches). This is the
same reasoning that made the RSS fetch a Job in Build Plan 01.

6. **Resolve the URL set to scrape** from config (see §10 #1 for the
   scoping decision): a bounded seed list of `docs.databricks.com`
   section roots, expanded via each domain's `sitemap.xml` (preferred over
   blind crawling), filtered to this installation's `tracked_clouds`
   (AWS → `docs.databricks.com/aws/...`, GCP →
   `docs.databricks.com/gcp/...`, Azure →
   `learn.microsoft.com/en-us/azure/databricks/...`).
7. **Respect robots.txt and rate limits.** Confirm scraping is permitted
   (see §10 #5), honor `robots.txt`, set a descriptive User-Agent, throttle
   requests (polite delay / bounded concurrency), and honor `Retry-After`
   on `429`/`503`.
8. For each URL, do a **conditional GET** using the stored `etag` /
   `last_modified` from `docs_scrape_state` (`If-None-Match` /
   `If-Modified-Since`). On `304`, skip — do not re-download or re-parse.
9. On `200`: clean the HTML → readable text (strip nav/chrome/boilerplate;
   convert to Markdown or plain text preserving heading structure), compute
   `content_hash` over the cleaned text, and — only if the hash changed
   from `docs_scrape_state` — write the cleaned page as a file to the
   Volume (path keyed by a stable, URL-derived name plus `content_hash`, so
   the chunk step can tell versions apart; see Part D's overwrite caveat).
10. **Upsert `docs_scrape_state`** for every URL touched: refreshed
    `etag`/`last_modified`/`content_hash`/`http_status`/`last_fetched_at`,
    and `last_error` on failure. A single URL's fetch/parse failure must
    **not** abort the whole run (capture per-URL errors; the run succeeds
    if most URLs succeed — mirrors Build Plan 01's non-throwing error
    capture).

### Part D — Chunk step: files → `docs_corpus` (`-infra`)

> **Decision — chunking is a Job step with `MERGE`, not an Autoloader
> LDP.** Build Plan 03 (Phase 2b) uses an Autoloader LDP to stream JSON
> findings into `research_log`; the symmetric choice here would be an
> Autoloader LDP over the Volume. But a low-frequency (weekly), re-scraped
> corpus hits Autoloader's overwrite semantics awkwardly (a re-fetched page
> is a *changed* file, which Autoloader does not re-ingest by default
> without `cloudFiles.allowOverwrites` or version-named paths), and the
> update is fundamentally a keyed upsert, not an append. A Job step that
> reads changed files and `MERGE`s into `docs_corpus` on `chunk_id` is
> simpler and idempotent. The Autoloader-LDP alternative (for symmetry with
> Phase 2b) is noted in §10 #4 for cross-review to weigh.

11. Read the page files written/updated this run (or, on a full rebuild,
    all files).
12. Split each page into **heading-aware chunks**, capturing `doc_title`,
    `section_path`, and `chunk_ordinal`; size each chunk under the
    embedding endpoint's token limit (§4).
13. Compute `chunk_id = sha2(url ‖ '#' ‖ chunk_ordinal, 256)` and
    `MERGE INTO docs_corpus` on `chunk_id` (update changed chunks, insert
    new ones). Set `content_hash`/`fetched_at` from the source page.
14. **Handle orphaned chunks.** If a page got *shorter* (fewer chunks than
    a prior version), the now-surplus `chunk_id`s from the old version must
    be **deleted** for that `url`, or the index will keep serving stale
    chunks. Delete `docs_corpus` rows for the page whose `chunk_ordinal >=`
    the new chunk count (or whose `content_hash` no longer matches the
    current one) as part of the same transaction/run. Call this out as a
    real correctness step, not an afterthought.

### Part E — Shared Vector Search endpoint + `docs_corpus` index (`-ai-tools`)

15. **Create (or reuse) the shared Storage-Optimized Vector Search
    endpoint.** Per `PROJECT-PLAN.md` §2 Phase 6, the docs-corpus index and
    Build Plan 03's `research_log` index **share one** Storage-Optimized
    endpoint. Whichever plan deploys first creates it **idempotently**; the
    other references it and must not create a second. See §6 for the
    coordination contract.
16. **Create the `docs_corpus` Delta Sync index** with **managed
    embeddings**, `TRIGGERED` pipeline type, `primary_key = chunk_id`,
    embedding source column `chunk_text`, and the pinned current
    managed-embedding endpoint (§2). Include `source_cloud`, `url`,
    `doc_title`, `section_path` in `columns_to_sync` so retrieval can
    filter/attribute results.
17. **Trigger a sync after each corpus refresh.** `TRIGGERED` indexes do
    not sync automatically — the refresh Job (Part C/D) must call
    `sync_index()` (or the equivalent) after the `MERGE`/delete completes,
    exactly as Build Plan 03's pipeline does for `research_log`. Confirm the
    current trigger mechanism against `databricks-vector-search`.
18. If the index is **not** a DAB-declarable resource (§3), implement the
    endpoint+index creation as the idempotent post-deploy setup job and
    have Part C/D's refresh call the sync.

### Part F — Refresh schedule + the corpus-optional contract with Build Plan 01

19. **Schedule the scrape+chunk+sync Job** (see §10 #2 — weekly proposed;
    the corpus is supplementary, so daily buys little). Use a standard
    Jobs cron schedule with an explicit `timezone_id` and
    `pause_status: UNPAUSED` (same conventions Build Plan 01 confirmed
    against `databricks-jobs`).
20. **Publish the index name as config** that Build Plan 01's Research
    Agent reads (a bundle variable / config table entry — coordinate the
    exact key with Build Plan 01's Part F step 13). Document the **contract
    explicitly**: the Research Agent's corpus lookup is *optional and
    best-effort* — if the index does not exist, is empty, or the query
    fails, Phase 2 logs and proceeds with live-fetched context only. This
    plan's incompleteness must never fail a Phase 2 run.

## 6. Explicit fan-out map

**Within this plan** — two tracks, largely parallel after the tables exist:

- **Track 1 — ingestion (`-infra`)**: Part A (Volume) ∥ Part B (tables)
  can be built in parallel (independent resources). Part C (scrape) depends
  on A + B. Part D (chunk/`MERGE`) depends on A + B + C's output. One
  implementer can own A→B→C→D as a unit.
- **Track 2 — serving (`-ai-tools`)**: Part E (endpoint + index) depends
  only on `docs_corpus` *existing* (Part B), and can be developed in
  parallel with Track 1 against an empty/seed table — but must **deploy**
  after `-infra` has run at least once (index is more useful once Part D
  has populated rows). A second implementer can own Track 2.
- Part F (schedule + config contract) is small config/docs; fold into
  whichever track finishes last.

**Across plans:**

- **Shared VS endpoint with Build Plan 03 (Phase 2b).** Both plans need the
  one Storage-Optimized endpoint. **Contract:** a single, idempotent
  "create endpoint if absent" helper, owned jointly; neither plan hard-codes
  creating a *second* endpoint. Because build order runs Phase 1 (this
  plan) before Phase 2b, **this plan is the natural creator**; Build Plan 03
  references it. Record the endpoint name as shared config.
- **Embedding-endpoint pin must match Build Plan 03.** Both indexes should
  use the same resolved managed-embedding endpoint; pin it once in shared
  config so the two indexes don't silently diverge.
- **Consumer: Build Plan 01 Part F step 13.** This plan *produces* the
  index that step reads. The interface is just "an index name + the
  graceful-degradation contract in Part F" — no code coupling.
- **Consumer: Build Plan 08 (Phase 6, stretch).** The KB-search UI / MCP
  server is a later consumer of this same index; nothing to build here for
  it, but keep `columns_to_sync` rich enough (url, title, section_path) to
  support attributed search results later.

## 7. Platform constraints to build against (to be cross-review-confirmed)

- **Delta Sync index requires Change Data Feed on the source table** —
  `docs_corpus` sets `delta.enableChangeDataFeed = true` (Part B). Confirm
  exact requirement/behavior against `databricks-vector-search`.
- **Managed embeddings need a Foundation Model embedding endpoint**; pin
  the *current* one, not a legacy GTE/BGE name (§2). Confirm the endpoint
  name and its **max input tokens** (drives chunk sizing, §4/Part D).
- **`TRIGGERED` indexes do not auto-sync** — an explicit `sync_index()`
  call after each refresh is required (Part E step 17).
- **One Storage-Optimized endpoint, shared** with Build Plan 03 (§6);
  a Delta Sync index maps 1:1 to one source table, so the two indexes stay
  separate even while sharing the endpoint (`PROJECT-PLAN.md` §2 Phase 6).
- **The Vector Search index may not be a DAB resource type** — verify
  `databricks bundle schema`; fall back to the idempotent post-deploy setup
  job (§3, Part E step 18).
- **UC Volume access via `grants`, not `permissions`** (§3).
- **Scraping is a Job, not an LDP** (no HTTP source; `http_request` in LDP
  rejected — §5 Part C).
- **`docs.databricks.com` / `learn.microsoft.com` scraping must respect
  robots.txt, ToS, and rate limits** (§5 Part C step 7; open item §10 #5).
- **`ai_prep_search` is deliberately not used here** (`PROJECT-PLAN.md`
  §2 Phase 1): its hard input contract is `ai_parse_document` output
  (binary docs — PDF/DOCX/images), not scraped HTML. Revisit only if the
  corpus source ever shifts to binary document files.

## 8. Validation checklist

- [ ] Scrape Job fetches a known docs URL and writes a cleaned file to the
      Volume.
- [ ] A second run against an **unchanged** page returns `304` (or an
      unchanged `content_hash`) and skips re-writing / re-chunking.
- [ ] A run against a **changed** page updates the file, re-chunks, and
      `MERGE`s without duplicating chunks (row count for that `url` is
      correct, not doubled).
- [ ] A page that **shrank** has its orphaned old chunks deleted (Part D
      step 14) — the index does not keep serving removed sections.
- [ ] `tracked_clouds` scoping works: an installation tracking only AWS does
      not scrape GCP/Azure docs.
- [ ] A single URL fetch/parse failure is captured in
      `docs_scrape_state.last_error` and does **not** abort the run.
- [ ] `docs_corpus` has CDF enabled; the Delta Sync index builds against it.
- [ ] The index is created on the **shared** Storage-Optimized endpoint
      (not a second endpoint) and uses the **current** managed-embedding
      endpoint (not a legacy model).
- [ ] After a refresh, `sync_index()` is called and a known query (e.g.
      "table update trigger") returns a relevant chunk with `url` /
      `section_path` populated.
- [ ] **Corpus-optional contract:** with the index absent or empty, a
      simulated Build Plan 01 corpus lookup degrades gracefully (logs +
      proceeds), does not raise.
- [ ] Chunk sizes are all within the embedding endpoint's token limit.
- [ ] Deploy order holds: `-infra` (Volume/tables/Job) deploys and runs
      before `-ai-tools` (endpoint/index) is created.

## 9. Acceptance criteria (Definition of Done)

- The `-infra` bundle creates the docs-corpus Volume, `docs_corpus`, and
  `docs_scrape_state`; the scrape+chunk Job runs on schedule and populates
  `docs_corpus` idempotently from the tracked clouds' docs.
- The `-ai-tools` bundle (or its post-deploy setup job) creates the shared
  Storage-Optimized endpoint (if absent) and the `docs_corpus` Delta Sync
  index with managed embeddings on the current embedding endpoint, synced
  after each refresh.
- A semantic query against the index returns relevant, attributed chunks.
- The index name is published as config for Build Plan 01, with the
  documented graceful-degradation contract, and a corpus-absent Phase 2 run
  is proven not to fail.
- All §8 checks pass. The changelog below records cross-review changes.

## 10. Open items carried forward (not blockers, tracked for later plans)

1. **URL / section scope (the biggest one).** Which
   `docs.databricks.com` sections seed the corpus? Options: (a) the docs
   areas matching the installed skills' domains (jobs, pipelines, vector
   search, apps, lakebase, dbsql, unity catalog, …); (b) a broader
   product-docs crawl bounded by URL prefix + sitemap; (c) start narrow and
   grow. **Proposed:** start with (a) — a bounded, config-driven seed list
   keyed to the skill set — since the corpus is only supplementary and a
   full-docs crawl is expensive and mostly unused. Needs the repo owner's
   call on the exact seed list.
2. **Refresh cadence.** **Proposed:** weekly (the corpus is supplementary;
   the RSS live-fetch is the freshness path). Confirm — daily is an option
   if cheap, but buys little given the Research Agent never relies on this
   corpus for same-day announcements.
3. **Scrape-state storage: Delta vs. Lakebase.** This plan puts the per-URL
   cursor in a Delta `docs_scrape_state` table (§4 decision) rather than
   Lakebase, on scale + non-App-facing grounds — diverging from §1c's RSS
   cursor placement. Confirm this is consistent with §1c's intent, or move
   it to Lakebase for uniformity.
4. **Chunk mechanism: Job `MERGE` vs. Autoloader LDP.** This plan chose a
   Job step with `MERGE` (§5 Part D decision) over an Autoloader LDP
   (which Build Plan 03 uses for findings). Confirm, or adopt the LDP for
   architectural symmetry if the overwrite-semantics concern is judged
   acceptable.
5. **Is scraping `docs.databricks.com` / `learn.microsoft.com` permitted?**
   Confirm ToS / robots.txt allow it and that a sitemap-driven, throttled
   fetch is acceptable. If not, fall back to an approved docs source (e.g.
   an official docs export / the `databricks-docs` skill's llms.txt index)
   before building Part C.
6. **Should the corpus also include the skills' own reference docs?** The
   installed skills carry `references/` markdown that is itself
   authoritative Databricks guidance. Indexing it alongside scraped docs
   could improve grounding — but risks the corpus echoing the very content
   the project is trying to *update*. Deferred; decide before broadening
   scope.
7. **Embedding-endpoint + shared-endpoint coordination with Build Plan 03.**
   The pinned embedding endpoint and the shared Storage-Optimized VS
   endpoint name must be identical across this plan and Build Plan 03.
   Record both as shared config; resolve who owns the "create if absent"
   helper (proposed: this plan, since it builds first — §6).
