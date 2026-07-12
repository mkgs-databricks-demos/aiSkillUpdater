# Build Plan 08 — Phase 6 (stretch): Knowledge-Base Search + MCP Server

**Source of truth:** `docs/PROJECT-PLAN.md` §2 Phase 6 (the search UI + the
App-hosted MCP server; the deliberate two-index design on one shared
Storage-Optimized endpoint; HYBRID query default), §1d (Genie Spaces + external
MCP tools are `-ai-tools` stretch resources; the App is `-app`), plus the
consumed contracts from Build Plan 02 (docs-corpus index), Build Plan 03
(`research_log` index + the VARIANT columns not synced into the index), and
Build Plan 04 (the App shell + Data Access Gate). This plan restates what's
needed to build Phase 6; it does not re-litigate decisions already locked
there — if something here seems to conflict with `PROJECT-PLAN.md`, that file
wins and this plan is stale.

**Cross-reviewed:** _pending_ — a first draft. An independent structural pass
(`pi`) and an independent Databricks/mechanics fact-check pass (`codex`) will
review this draft; every BLOCKING finding from both will be folded into a
revised version, with a changelog at the bottom.

**What this phase is, in one sentence:** it exposes the knowledge this project
has accumulated (`research_log` + the docs corpus, already indexed in Vector
Search) as two new *consumers* of that same data — a search UI inside the
Review App, and a Databricks App-hosted MCP server so other agents can query it
programmatically — without changing anything upstream.

**Why this is a stretch phase and last (design rationale):** it is purely
*additive* — a new reader of data Phases 1–3 already produce and index, not a
dependency of the core RSS→research→review→install loop (`PROJECT-PLAN.md` §2
Phase 6). Nothing else needs it to function, so it is built only after Phases
1–5 are solid.

> **Decisions made in this plan that `PROJECT-PLAN.md` does not pin down —
> and this phase has more genuine unknowns than any other.** Phase 6 names the
> *what* (search UI + MCP server, two indexes/one endpoint, HYBRID) but leaves
> the *how* almost entirely open, and the **MCP-server hosting mechanism is the
> single biggest unknown in the whole project** — "a Databricks App-hosted MCP
> server" is one phrase in the source doc and needs real build-time
> investigation before committing. This plan makes explicit, flagged choices
> for: **how the MCP server is actually hosted** — custom-in-App vs. a
> Databricks-managed Vector Search MCP server vs. a model-serving front (§5
> Part B, §7, §11 #1); **MCP auth + per-installation scoping** for external
> agents (§5 Part B, §11 #2); the **HYBRID query API** specifics (§5 Part A,
> §11 #3); the **search-UI Vector Search access path** from the App and how it
> relates to the Data Access Gate (§5 Part A, §11 #4); whether to include a
> **Genie Space** as an alternative NL surface (§5 Part C, §11 #5); and the
> **MCP tool surface** exposed (§4, §11 #6). Each is consolidated in §11 for
> the repo owner and cross-review to confirm or override — and given the MCP
> hosting uncertainty, a scoped build-time spike is expected before Part B.

---

## 1. Objective

Make the accumulated knowledge searchable — for humans and for other agents.
When this plan is done:

1. A **search UI in the Review App** lets a human query `research_log` (and the
   docs corpus) semantically + by keyword — "what's the latest guidance on X,"
   "which skill covers Y" — with results linking to the full `research_log`
   entry (summary, citations, availability) and the relevant skill.
2. `research_log` search defaults to **HYBRID** (semantic ANN + keyword/BM25),
   since queries frequently contain exact skill names, feature names, and
   compliance terms (HIPAA, PCI-DSS, FedRAMP) that need literal matching
   alongside semantic recall.
3. A **Databricks App-hosted MCP server** exposes the same knowledge
   programmatically so other agents can query it (a small tool surface — e.g.
   "search guidance," "which skill covers X," "get entry by id").
4. Both consumers reuse the **existing two indexes on the shared
   Storage-Optimized endpoint** (docs corpus from Build Plan 02, `research_log`
   from Build Plan 03) — no new index, no merged index, no second endpoint.
5. Everything is **per-installation-scoped** and read-only against the indexed
   data.

**Out of scope (owned elsewhere or explicitly additive-only):** creating or
changing the indexes (Build Plans 02/03 own them — this plan only queries);
the App shell/identity/Data-Access-Gate plumbing (Build Plan 04 — this plan
adds a surface inside it); anything in the core loop (this phase is a new
consumer, never a dependency).

## 2. Prerequisites

- **Build Plans 02 + 03 built and populated** — the docs-corpus index and the
  `research_log` index exist on the shared Storage-Optimized endpoint, with
  real content to search. (Build Plan 03 §4 deliberately did **not** sync the
  VARIANT columns `citations`/`availability` into the index, so full-detail
  results are read from the `research_log` Delta row by `research_log_id` —
  this plan follows that.)
- **Build Plan 04 built** — the App shell hosts the search UI, and (per §11 #1's
  outcome) very possibly the MCP server too; the Data Access Gate and
  per-installation scoping are Build Plan 04's.
- **A build-time investigation of MCP hosting on Databricks** (§11 #1) — this
  should happen *before* Part B is scoped, given how open the mechanism is.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

| Resource | Bundle | Notes |
|---|---|---|
| Search UI | `-app` | A surface inside Build Plan 04's App. |
| MCP server | *open — §11 #1* | If custom-hosted in the App → `-app`; if a Databricks-managed Vector Search MCP server or a model-serving front → `-ai-tools` (alongside the indexes/endpoints it fronts). The hosting decision determines the bundle. |
| Genie Space (optional, §5 Part C) | `-ai-tools` | Per §1d, Genie Spaces are `-ai-tools` stretch resources (data-must-exist-first). Only if pursued (§11 #5). |
| The two Vector Search indexes + shared endpoint | *not here* | Owned by Build Plans 02/03 (`-ai-tools`); consumed read-only. |

**No new indexes, endpoints, Delta tables, or Lakebase tables.** This phase
adds only query surfaces over existing resources (plus possibly a small amount
of MCP-server state if the chosen hosting needs it — decided with §11 #1).

## 4. What's exposed & query model

Two consumers over the **same** two indexes (shared Storage-Optimized endpoint):

**(a) Search UI (in the App):**
- Query `research_log` via its index in **HYBRID** mode (§5 Part A); optionally
  the docs-corpus index too.
- Render results with a link to the full `research_log` entry (reading
  `summary`, and the VARIANT `citations`/`availability` from the Delta row by
  `research_log_id`, since those aren't in the index — Build Plan 03 §4) and to
  the classified skill.

**(b) MCP server (App-hosted, §11 #1):** a small tool surface (§11 #6), e.g.:
- `search_guidance(query, filters?)` — HYBRID search over `research_log`,
  returns ranked entries (id, summary snippet, label, links).
- `which_skill_covers(query)` — returns the most relevant skill(s)/label(s).
- `get_entry(research_log_id)` — full entry detail (summary + citations +
  availability from the Delta row).
- (optional) `search_docs(query)` — over the docs-corpus index.
All scoped per `installation_id`; all read-only.

## 5. Task breakdown

### Part A — Search UI + HYBRID query (`-app`, extends Build Plan 04)
1. Add a **search surface** to the Review App that issues a **HYBRID** Vector
   Search query against the `research_log` index (semantic ANN + keyword/BM25),
   per `PROJECT-PLAN.md` §2 Phase 6 — confirm the exact HYBRID query API /
   `query_type` parameter against current Vector Search docs (§11 #3).
2. Establish the **App→Vector Search access path** (§11 #4): how the App issues
   the VS query (SDK/REST via the App SP). Note this is a *different* access
   than Build Plan 04's Data Access Gate (which governs warehouse-vs-Lakebase
   reads of `research_log` rows) — a VS index query is its own path; clarify how
   the two coexist (the Gate for row detail, VS for ranked retrieval).
3. Render results: ranked entries with a link to the full `research_log` entry;
   read `citations`/`availability` from the Delta row by `research_log_id` (not
   the index — Build Plan 03 §4). Scope every query by `installation_id`.

### Part B — MCP server (hosting decided by §11 #1; likely `-app`)
4. **Decide the hosting mechanism first (§11 #1 — do a build-time spike):**
   - **custom MCP server in the App** — the App serves an MCP endpoint
     (JSON-RPC/SSE or streamable HTTP) directly;
   - **a Databricks-managed Vector Search MCP server** — if the platform offers
     a managed MCP surface over a VS index, front the `research_log` index with
     it;
   - **a model-serving front** — expose via a serving endpoint
     (`databricks-model-serving` is listed as an optional dep only "if fronting
     via an endpoint," `PROJECT-PLAN.md` §3).
   This is the biggest unknown in the project; confirm what Databricks actually
   supports before building (§7).
5. Implement the **tool surface** (§4 / §11 #6): `search_guidance`,
   `which_skill_covers`, `get_entry`, optional `search_docs` — thin wrappers
   over the same HYBRID query + `research_log` row reads Part A uses.
6. **Auth + per-installation scoping for external agents (§11 #2):** define how
   a calling agent authenticates to the MCP server and how its queries are
   scoped to the right installation's data. Do not expose cross-installation
   data; do not expose write operations (read-only surface).

### Part C — Optional: Genie Space as an alternative NL surface (`-ai-tools`, §11 #5)
7. Optionally stand up a **Genie Space** over `research_log` as a
   natural-language query alternative to the search UI / MCP server. Per §1d
   this is an `-ai-tools` stretch resource (data-must-exist-first). Explicitly
   optional; do not block the phase on it — decide in §11 #5 whether it earns
   its keep alongside the search UI + MCP server or is redundant.

## 6. Cross-plan contracts (shared config — do not re-decide here)

- **The two indexes + shared Storage-Optimized endpoint** (Build Plan 02
  docs-corpus, Build Plan 03 `research_log`): consumed read-only. **Two separate
  indexes is deliberate** (`PROJECT-PLAN.md` §2 Phase 6 — different source
  tables/schemas/cadences; a Delta Sync index maps 1:1 to one source table), so
  this phase queries both, never merges them.
- **`research_log` VARIANT columns are not in the index** (Build Plan 03 §4):
  full `citations`/`availability` detail is read from the Delta row by
  `research_log_id`; the index carries only the filterable scalars + the
  `summary` embedding. Both consumers here follow that split.
- **The App shell + Data Access Gate + per-installation scoping** are Build Plan
  04's; this phase adds surfaces inside that App and reuses its scoping.
- **`databricks-model-serving` is an optional dependency only** "if fronting
  [the MCP server] via an endpoint" (`PROJECT-PLAN.md` §3) — not committed to
  unless §11 #1 chooses that hosting.

## 7. Platform constraints to build against (to be cross-review-confirmed)

- **MCP-server hosting on Databricks is the top unknown (§5 Part B, §11 #1).**
  Confirm what's actually supported *now*: whether Databricks Apps can host a
  custom MCP endpoint, whether there's a managed Vector Search MCP server, and
  whether a model-serving front is the idiomatic path. Do **not** assume a
  mechanism — a build-time spike precedes committing to one.
- **HYBRID query support (§5 Part A, §11 #3):** confirm the Vector Search HYBRID
  query API (the `query_type`/hybrid parameter and any reranking options)
  against current docs — this was cross-review-guidance in `PROJECT-PLAN.md` §2
  Phase 6, confirm the exact call.
- **Shared endpoint capacity:** two indexes already share one Storage-Optimized
  endpoint; adding query load (UI + MCP) should be checked against the
  endpoint's throughput expectations, though read query load is typically light.
- **MCP is read-only here** — the server exposes no write/mutation tools; it
  cannot accept/apply/merge anything (that stays in the human-gated Build Plan
  04/05 flow — cf. Build Plan 07's guardrail).
- **Per-installation scoping** must hold for both the UI and the MCP server —
  no cross-installation data leakage through either surface.

## 8. Explicit fan-out map

**Within this plan:**
- **Part A (search UI)** is self-contained and the safest to build first — it
  reuses Build Plan 04's App + the existing index, just adds a HYBRID query
  surface.
- **Part B (MCP server)** is gated on the §11 #1 hosting spike — do that spike
  before scoping Part B; then the tool surface (step 5) reuses Part A's query
  logic, and auth/scoping (step 6) is the remaining real work.
- **Part C (Genie Space)** is optional and independent — pursue only if §11 #5
  says it earns its keep.

**Across plans:**
- **Blocked on Build Plans 02 + 03** (the indexes) and **Build Plan 04** (the
  App shell). Purely additive — nothing downstream depends on this phase.
- **Note**: the search UI, the MCP server, and any Genie setup are real code /
  config — when built, each is an `implement` task for a coding sub-agent → its
  own PR → cross-review → human merge; the MCP hosting spike is an `explore`
  task first.

## 9. Validation checklist

- [ ] A HYBRID `research_log` search returns sensible results for a query mixing
      an exact term (e.g. `HIPAA`, a skill name) and a semantic phrase — verify
      the literal term is actually matched, not just semantically approximated.
- [ ] Search results link to the full `research_log` entry, and the entry detail
      reads `citations`/`availability` from the Delta row (not the index).
- [ ] The MCP server (once §11 #1 hosting is chosen) responds to
      `search_guidance` / `which_skill_covers` / `get_entry` with correct,
      per-installation-scoped, read-only results.
- [ ] The MCP server exposes **no** write/mutation tool.
- [ ] Both surfaces scope strictly by `installation_id` — a query from one
      installation never returns another's data.
- [ ] Both indexes are queried as-is on the shared endpoint — no new index, no
      merge, no second endpoint created.
- [ ] (If pursued) the Genie Space answers NL questions over `research_log`
      consistently with the search UI.
- [ ] The MCP-hosting spike (§11 #1) produced a confirmed, supported mechanism
      before Part B was implemented — not an assumed one.

## 10. Acceptance criteria (Definition of Done)

- The Review App has a working HYBRID `research_log` search surface, results
  link to full entries, scoped per installation.
- An App-hosted (or otherwise Databricks-supported, per §11 #1) MCP server
  exposes a small read-only tool surface over the same knowledge, authenticated
  and per-installation-scoped, with no write tools.
- Both consumers reuse the existing two indexes on the shared endpoint; nothing
  upstream is changed.
- The optional Genie Space is either built (if §11 #5 chose to) or explicitly
  deferred.
- The MCP-hosting mechanism was confirmed by a build-time spike, not assumed.
- Every §11 open item is either resolved and recorded here, or explicitly
  carried forward with its downstream owner.

## 11. Open items carried forward (decisions to confirm in cross-review)

1. **MCP-server hosting mechanism (§5 Part B, §7) — the biggest unknown in the
   project.** Custom MCP endpoint hosted in the Databricks App vs. a
   Databricks-managed Vector Search MCP server vs. a model-serving front.
   Requires a build-time spike to confirm what Databricks actually supports
   *now* before committing; the choice also sets the MCP server's bundle (§3).
2. **MCP auth + per-installation scoping for external agents (§5 Part B step
   6).** How a calling agent authenticates to the MCP server and how its queries
   scope to one installation's data; read-only enforcement.
3. **HYBRID query API specifics (§5 Part A step 1).** Confirm the Vector Search
   `query_type`/hybrid parameter and any reranking options against current docs.
4. **App→Vector Search access path vs. the Data Access Gate (§5 Part A step
   2).** How the App issues the VS query (SDK/REST via App SP) and how that
   coexists with Build Plan 04's warehouse-vs-Lakebase Data Access Gate (the
   Gate governs row-detail reads; VS is a separate retrieval path).
5. **Genie Space — build it or not (§5 Part C).** Whether a Genie Space over
   `research_log` earns its keep as a third NL surface alongside the search UI +
   MCP server, or is redundant. Optional, non-blocking.
6. **MCP tool surface (§4).** Confirm the exact tools (`search_guidance`,
   `which_skill_covers`, `get_entry`, optional `search_docs`) and their
   input/output shapes.

## 12. Changelog

- **Initial draft** (pre-cross-review): drafted from `PROJECT-PLAN.md` §2
  Phase 6 and §1d, and the consumed index/App contracts in Build Plans 02, 03
  (VARIANT-not-in-index split), and 04 (App shell + Data Access Gate). Framed
  the phase as purely additive — two new read-only consumers (search UI + MCP
  server) over the existing two-index/shared-endpoint setup, never a new index
  or a dependency of the core loop. Made explicit flagged decisions for the six
  open items in §11 — foregrounding that the **MCP-server hosting mechanism is
  the single biggest unknown in the project** and needs a build-time spike
  before Part B is scoped (§5 Part B / §7 / §11 #1), rather than assuming an
  App-hosting approach that may not be supported as written.
