# Build Plan 04 — Phase 3: Review App (the human gate + markdown editor)

**Source of truth:** `docs/PROJECT-PLAN.md` §2 Phase 3 (the Review App
design), §1a (bring-your-own-repo, starter packs, cloud/region config), §1c
(Lakebase vs. Delta boundary — review-queue state, overrides, taxonomy
buckets, synced-table caveats, deploy-first schema ownership), §1d
(three-bundle split — the App lives in `-app`), §5 (versioning frontmatter
the accept/save flow must preserve and increment). This plan restates what's
needed to build Phase 3; it does not re-litigate design decisions already
locked there — if something here seems to conflict with `PROJECT-PLAN.md`,
that file wins and this plan is stale.

**Cross-reviewed:** an independent structural pass (`pi`) and an independent
Databricks/Apps-mechanics fact-check pass (`codex`) both reviewed a first draft
of this plan; every BLOCKING finding from both is folded into the version
below. See the changelog at the bottom of this file.

**What this phase is, in one sentence:** a Databricks App (AppKit + React)
that reads the `research_log` entries produced by Build Plan 03 and presents a
human review queue — accept / reject / edit-then-accept each proposed skill
change, override its classification, browse and hand-edit any skill file — and
on accept, opens a pull request against this installation's configured skills
repo (§1a) using the GitHub App installation token minted by Build Plan 00's
flow.

**Why this phase is the trust boundary (design rationale, not re-litigated
here):** nothing this project generates ever touches a user's skills without
passing through this queue. The AI pipeline (Build Plans 01–03) only ever
*proposes*; a human accepts here, and only an accept opens a PR. The PR — not
an auto-merge — is the deliverable, consistent with the whole project's
human-in-the-loop posture. See `PROJECT-PLAN.md` §2 Phase 3.

> **Decisions made in this plan that `PROJECT-PLAN.md` does not pin down.**
> Phase 3 in the source doc specifies the *what* (a review queue over
> `research_log`, a markdown editor, classification override that re-triggers
> generation, an accept→PR flow, a starter-pack "check for latest" action into
> the same queue) but leaves several *hows* open. This plan makes explicit,
> flagged choices for: the **review-queue Lakebase data model** (§4, §11 #1);
> **the entry point the classification-override re-trigger actually calls**
> into Build Plan 01's Research Agent Job — a genuine cross-plan gap, since
> that Job has no single-entry regeneration mode today (§5 Part D, §6, §11 #2);
> **where the §5 frontmatter version bump logic lives** on accept/save (§5
> Part E, §11 #3); **how a single review queue holds both `research_log`-origin
> and starter-pack-diff-origin items** (§4, §5 Part F, §11 #4); the **SQL
> warehouse resource** the Data Access Gate reads through (§3, §11 #5); and how
> the App **scopes every read/write by `installation_id`** given the
> one-App-vs-one-App-per-installation question inherited from Build Plan 01
> (§5 Part A, §11 #6). Each discretionary choice is consolidated in §11 for the
> repo owner and cross-review to confirm or override.

---

## 1. Objective

Build the Databricks App that is the project's human review surface. Concretely, when this plan is done:

1. A reviewer opens the App and sees a **review queue**: one card per pending
   proposal, each showing the AI summary (with footnote citations), the
   proposed diff (or drafted new-file content), the classified target skill,
   and the extracted cloud/compliance availability.
2. Per card, the reviewer can **accept**, **reject**, or **edit-then-accept**
   (hand-edit the proposed markdown before accepting).
3. Per card, the reviewer can **override the classification** — reassign to a
   different existing skill, to `"New Feature"`, or type a brand-new bucket
   name — which **re-triggers generation** under the corrected label rather
   than just relabeling the stale proposal.
4. **Accept** opens a **pull request** against this installation's configured
   skills repo (§1a), writing the accepted content with correctly
   incremented §5 versioning frontmatter, and exposes an (optional) hook for
   Build Plan 05's Local Installer.
5. A **Skill Library browser** lets the reviewer view **any** skill file at any
   time (not just ones with a pending proposal), hand-edit it, and save the
   edit back as a PR.
6. A **"Check for latest from `<starter pack>`"** action (per-skill or a full
   sweep) diffs the installation's current skills against its configured
   starter pack's live source repo and feeds the results into the **same**
   review queue.

**Out of scope for this plan (owned elsewhere):** the actual local file-copy
mechanism (Build Plan 05 — this plan only exposes the accept-time hook and the
"apply locally" affordance); the three-way CLI-drift conflict *detection*
(Build Plan 04 = Phase 3 owns only the App shell that Build Plan 06 / Phase 4
will later render its three-way conflict UI inside); the KB/MCP server (Build
Plan 08); the maintainer-only upstream-PR path (`PROJECT-PLAN.md` §6 open
question #4, deliberately deferred nice-to-have — this plan must not surface it
to non-maintainer users, but does not build it; see §11 #8 for the gate this
plan must design around).

## 2. Prerequisites

- **Build Plan 00 fully built and deployed.** This plan hard-depends on:
  - The Lakebase database/schema owned by the App's service principal (the
    review-queue tables in §4 are created in that same schema).
  - `installations`, `tracked_clouds`, `tracked_regions`, `starter_packs`
    tables (§4 references, does not recreate).
  - The GitHub App registered + the per-installation `github_installation_id`
    stored, and the **server-side installation-access-token minting** the
    accept/PR flow calls (Build Plan 00 §5 Part A/B). The stable secret key
    names (`github-app-id`, `github-app-private-key-pem`, etc.) are consumed
    here, not re-declared.
- **Build Plan 03 fully built and deployed**, with `research_log` populated
  with a handful of real rows (Build Plan 03 §10 explicitly names this plan as
  the dependent that needs real rows). This plan reads `research_log` and the
  frozen schema in Build Plan 03 §4; the `research_log_id` content-hash is the
  foreign key the review-queue state points at.
- **Build Plan 01's Research Agent Job exists** (its `-infra` Job) — the
  classification-override re-trigger calls into it (§5 Part D). See §11 #2 for
  the entry-point contract that must be reconciled with Build Plan 01 first.
- **A SQL warehouse** the App reads `research_log` through (Data Access Gate,
  §3 / §5 Part B). Existence/placement is an open item (§11 #5).
- **Deploy-first, then dev** (`PROJECT-PLAN.md` §1c): the App's SP must have
  deployed (creating/owning its Lakebase schema) before any local dev against
  the review-queue tables, or dev fails with Postgres `42501`, not a friendly
  message. Call this out in the runbook.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

| Resource | Bundle | Notes |
|---|---|---|
| The Databricks App resource itself | `-app` | Per §1d, the App is `-app`, deployed last (after `-infra` and `-ai-tools` are live). Includes the explicit post-deploy "start the App" step §1d notes (a bundle deploy alone can leave it stopped). |
| Review-queue Lakebase tables (§4) | `-app` *(proposed)* | They are the App's own operational state and are created in the App SP's Lakebase schema. Placing their DDL/migration with the App keeps schema ownership with the SP that owns the schema (§1c). Flagged in §11 #1 — could instead live in `-infra` alongside Build Plan 00's Lakebase tables; decide with the Lakebase migration mechanism (§11 #7). |
| SQL warehouse for `research_log` reads | *open — §11 #5* | Whether a new warehouse is declared (and in which bundle) or an existing one is referenced by the App's `sql_warehouse` resource. |
| App SP `SELECT` on `research_log` | `-infra` | Already listed in Build Plan 03 §6 as an `-infra` grant, targeting the App SP created in `-infra` (Build Plan 00). This plan consumes it; it does not re-grant. |
| GitHub App secret scope | `-infra` | Created by Build Plan 00; this plan reads the credentials at runtime to mint installation tokens. Not re-declared. |

## 4. Data model (Lakebase review-queue state)

All of these live in the App SP's Lakebase schema (§1c routes review-queue
state, classification overrides, and user-created taxonomy buckets to
Lakebase, explicitly **not** Delta). `research_log` itself stays in Delta and
is **read-only** to this App — the review decision is *never* written back into
`research_log` (Build Plan 03 §4 states this: `research_log.status` is
read-only finding-origin lineage, not the review decision).

> **These are proposed (§11 #1)** — the shape below is this plan's design call,
> to be confirmed/overridden in cross-review.

### `review_items`
One row per reviewable proposal. A proposal originates **either** from a
`research_log` row **or** from a starter-pack diff (§5 Part F), so the
`research_log_id` FK is **nullable** and an `origin` discriminator distinguishes
them.

| Column | Type | Notes |
|---|---|---|
| `review_item_id` | uuid, PK | This App's own id. |
| `installation_id` | FK → `installations` | Every read/write in the App is scoped by this (§5 Part A, §11 #6). |
| `origin` | enum(`research_log`, `starter_pack_diff`) | Discriminates the two sources that feed the one queue. |
| `research_log_id` | text, nullable | FK-by-value to `research_log.research_log_id` (Delta, cross-store — not a DB-enforced FK). Set when `origin = research_log`; null for pack diffs. |
| `starter_pack_id` | FK → `starter_packs.pack_id`, nullable | Set when `origin = starter_pack_diff`. |
| `target_skill_path` | text, nullable | The skill file this proposal would change; null for a not-yet-named New Feature. |
| `proposed_content_ref` | text | Where the proposed markdown/diff is held for review — see §5 Part C (do **not** store large blobs inline if a pointer/verbatim-from-`research_log` read suffices; pack-diff content needs its own storage). Resolve exact mechanism in cross-review (§11 #1). |
| `decision` | enum(`pending`, `accepted`, `rejected`) | The human click-state — the thing §1c says lives in Lakebase, never in `research_log`. |
| `decision_note` | text, nullable | Optional reviewer comment. |
| `edited` | boolean | True if the reviewer hand-edited before accepting (edit-then-accept). |
| `pr_url` | text, nullable | Set once the accept flow opens a PR (§5 Part E). |
| `pr_state` | enum(`none`, `open`, `merged`, `closed`), default `none` | So the queue can show PR status and avoid re-opening duplicates. |
| `created_at` / `updated_at` | timestamptz | |

### `classification_overrides`
One row per human override of a proposal's classification. An override
**re-triggers generation** (§5 Part D), so this records the corrected label and
links to the regeneration run.

| Column | Type | Notes |
|---|---|---|
| `override_id` | uuid, PK | |
| `installation_id` | FK | |
| `research_log_id` | text | The entry being reclassified. |
| `original_label` | text | The model's label. |
| `override_label` | text | The human-confirmed label — an existing skill, `"New Feature"`, or a new bucket name (which also inserts into `taxonomy_buckets`). |
| `regeneration_run_id` | text, nullable | The Job run id returned by the fire-and-poll re-trigger (§5 Part D). |
| `regeneration_status` | enum(`triggered`, `running`, `succeeded`, `failed`) | Polled from the Job run. |
| `created_at` | timestamptz | |

### `taxonomy_buckets`
User-created classification buckets beyond the skill set + `"New Feature"`.
Build Plan 01 §5 Part F step 15 already reads these when building the
`ai_classify` label set ("plus any user-created bucket names already recorded
from prior overrides (Build Plan 04)") — this is the table it reads.

| Column | Type | Notes |
|---|---|---|
| `bucket_id` | uuid, PK | |
| `installation_id` | FK | Buckets are per-installation (taxonomies differ per configured repo). |
| `bucket_name` | text | UNIQUE per `installation_id`. Becomes a permanent classification label, and eventually a new skill directory. |
| `created_by` | text | Reviewer identity (§5 Part A). |
| `created_at` | timestamptz | |

**Cross-store integrity note:** `review_items.research_log_id` and
`classification_overrides.research_log_id` point at Delta rows, so there is no
DB-level FK. The stability of `research_log_id` (Build Plan 03 §11 #1 —
content-addressed, stable across re-ingestion) is what makes these
cross-store references safe; call out that a re-ingested finding keeps the same
id, so a pending review item does not dangle.

## 5. Task breakdown

### Part A — App shell, identity, and per-installation scoping (`-app`)
1. Scaffold the Databricks App as **AppKit + React** (locked in
   `PROJECT-PLAN.md` §2 Phase 3; the `databricks-apps-python` Python-framework
   fallback is explicitly *not* chosen here). Invoke the `databricks-apps`
   skill before scaffolding, per its own guidance.
2. Establish the **user identity** model: Databricks Apps forwards the calling
   user's identity; the App uses it to stamp `taxonomy_buckets.created_by`,
   `review_items.decision`, etc. **GitHub operations do NOT use the end user's
   identity** — they use the App SP's minted installation token (§5 Part E).
   State this split explicitly.
3. Establish **per-installation scoping**: every query in §5 Parts B–G filters
   by `installation_id`. How the App resolves "which installation am I" depends
   on the one-App-vs-one-App-per-installation decision inherited from Build
   Plan 01 §10 — write all data access to take `installation_id` as a
   parameter so either model works (§11 #6).

### Part B — Read `research_log` through the Data Access Gate (`-app` + `-infra` grant)
4. Read `research_log` for the queue via the **warehouse Analytics path**
   (`config/queries/` + `useAnalyticsQuery`), **not** a custom endpoint running
   a raw `SELECT` — `PROJECT-PLAN.md` §2 Phase 3 locks this (the raw-SELECT
   pattern is explicitly discouraged by `databricks-apps`). The App SP's
   `SELECT` grant on `research_log` comes from `-infra` (Build Plan 03 §6).
5. Only if the Analytics path proves too slow for the queue in practice, fall
   back to a **Lakebase synced table** (read-only mirror of `research_log`,
   §1c) — with §1c's caveats: synced tables are read-only in Postgres (writes
   corrupt the sync), the App SP needs an explicit GRANT to read them
   (`CAN_CONNECT_AND_CREATE` alone is insufficient), row/column ACLs are not
   propagated, and creation is via `databricks postgres create-synced-table`
   (the `databricks-lakebase` skill notes DAB support for autoscaling synced
   tables is "not yet available" — treat "not a DAB resource" as a
   version-sensitive claim to re-check against live docs at build time, not as
   timeless). Do **not** reach for this up front.
6. Join each `research_log` row to its `review_items` row (by
   `research_log_id`, creating a `pending` `review_items` row on first sighting
   if absent) so the queue reflects Lakebase-owned decision state layered over
   the Delta-owned finding.

### Part C — Review queue UI: cards, diff, citations, markdown editor (`-app`)
7. Render **one card per pending item**, showing: the `summary` markdown with
   its inline `[n]` citations rendered as **footnotes linking out to source**
   (read `citations` VARIANT from the `research_log` Delta row by
   `research_log_id` — Build Plan 03 deliberately does **not** sync the VARIANT
   columns into the VS index, so read them from the row); the classified
   `label` / `target_skill_path`; and the extracted `availability`
   (cloud + region-scoped compliance).
8. Render the **proposed change**. `research_log.proposed_diff` may be (a) a
   unified diff, (b) drafted new-file content (New Feature / new bucket, where
   `target_skill_path` is null), or (c) null — the card must handle all three
   distinctly (diff view, new-file preview, or "no diff — classification only"),
   per Build Plan 03 §4.
9. Provide a **markdown editor** for edit-then-accept: the reviewer edits the
   proposed text in-card before accepting; setting `review_items.edited = true`.
   The edited text (not the model's original) is what the accept flow commits.
10. Provide **accept / reject / edit-then-accept** controls writing
    `review_items.decision` (+ `decision_note`) in Lakebase. Never write the
    decision back into `research_log`.

### Part D — Classification override + generation re-trigger (`-app` → Build Plan 01 Job)
11. Make the classified `label` editable per card: reassign to another existing
    skill, to `"New Feature"`, or type a **new bucket name** (which inserts a
    `taxonomy_buckets` row). Write a `classification_overrides` row.
12. On override, **re-trigger generation under the corrected label** — do not
    just relabel the stale proposal (`PROJECT-PLAN.md` §2 Phase 3). This call:
    - uses **AppKit's `jobs()` plugin**, never a bespoke Jobs-SDK custom
      endpoint (`PROJECT-PLAN.md` §2 Phase 3, cross-reviewed vs.
      `databricks-apps`);
    - is **fire-and-poll** (`runNow`, then poll run status into
      `classification_overrides.regeneration_status`), never synchronous
      run-and-wait — the Apps reverse proxy enforces a hard **120 s
      per-request timeout**, well under a full research/classify/extract/diff
      regeneration;
    - targets **Build Plan 01's Research Agent Job** — **but that Job has no
      single-entry-by-`research_log_id`-under-override-label run mode today.**
      This is a real cross-plan gap: see §6 and §11 #2 for the entry-point
      contract that must be added to Build Plan 01 (a parameterized run mode:
      `mode=regenerate_single`, `research_log_id`, `override_label`,
      `installation_id`) before this step is buildable.
13. When regeneration completes, Build Plan 03's pipeline lands a new/updated
    `research_log` row (same `research_log_id`, stable per its §11 #1), and the
    card refreshes to show the diff grounded in the human-confirmed label.

### Part E — Accept → open a PR to the configured skills repo (`-app` → GitHub)
14. On **accept**, open a **pull request** against this installation's
    configured skills repo (resolve `installations.github_repo_owner` /
    `github_repo_name` / `default_branch`, §1a / Build Plan 00 §4):
    - Mint a short-lived **installation access token** server-side from the
      GitHub App credentials + this installation's `github_installation_id`
      (Build Plan 00 §5 Part A/B). No PAT, no end-user GitHub identity. Whether
      the App backend mints directly or via a shared helper is the same
      question Build Plan 01 §10 left open (§11 #2).
    - Write the accepted content (edited text if `edited`, else the proposed
      content) to `target_skill_path` on a new branch, commit, open a PR. Never
      merge — the human merges (project-wide rule). Store `pr_url` / `pr_state`
      on the `review_items` row.
15. **Increment the §5 versioning frontmatter** on the written file — this is
    the App backend's responsibility (§11 #3): bump `metadata.version` (append
    the project's own `+au.N` increment segment), set `updated_at`, set
    `last_research_log_id` to this entry's `research_log_id`, and preserve
    `base_cli_version` / `base_skills_repo_ref` untouched. For a brand-new skill
    file, initialize the frontmatter (including `availability` from the
    `research_log` row). Do this consistently for both accept and Part G's
    editor-save, so every PR this App opens carries correct version metadata.
16. Expose the **optional Local Installer hook** (`PROJECT-PLAN.md` §2 Phase 3
    accept fork "(b)"): accept always opens the PR; it *may* also offer "apply
    to this machine now." The actual apply mechanism is **Build Plan 05** — this
    plan only defines the hook/affordance and defers the implementation, so it
    must not hardcode any per-agent path logic here.

### Part F — Starter-pack "check for latest" into the same queue (`-app` → GitHub)
17. Add a **"Check for latest from `<starter pack>`"** action in the Skill
    Library browser (§1a — per-skill or full-sweep). Resolve the pack via
    `installations.starter_pack_id` (live FK) → `starter_packs`
    `source_repo_owner` / `source_repo_name` / `source_ref` (Build Plan 00 §4 —
    a live reference, not a one-time snapshot).
18. Fetch the pack's current skill content at `source_ref`, diff against the
    installation's current skill(s), and for each difference create a
    `review_items` row with `origin = starter_pack_diff` (nullable
    `research_log_id`, set `starter_pack_id`). These flow through the **exact
    same** queue, diff view, and accept→PR path as `research_log` proposals —
    no second UI (`PROJECT-PLAN.md` §1a / §2 Phase 3). No re-trigger applies
    (there is no AI generation step for a pack diff).

### Part G — Skill Library browser: view/edit any file, save as PR (`-app` → GitHub)
19. List all skills in the configured repo; view any `SKILL.md` (and its
    `references/`) **whether or not** it has a pending proposal
    (`PROJECT-PLAN.md` §2 Phase 3).
20. Allow **hand-edit + save**, which opens a PR exactly like an accept (§5
    Part E, including the frontmatter bump in step 15) — reading current content
    and writing the edit back to the configured repo via the installation
    token.
21. Leave an **extension point** for Build Plan 06 / Phase 4's three-way
    CLI-drift conflict UI to render inside this browser later (do not build the
    detection or the three-way diff here — just don't foreclose it).

## 6. Cross-plan contracts (shared config — do not re-decide here)

- **`research_log` schema + `research_log_id`** (Build Plan 03 §4): consumed
  read-only. `research_log_id` = `rl_` + stable hash of
  (`installation_id`, `rss_guid`, `label`), **stable across re-ingestion** — the
  property that makes `review_items` / `classification_overrides` cross-store
  references safe.
- **Lakebase `installations` / `starter_packs`** (Build Plan 00 §4): consumed,
  not recreated. `installations.default_branch` was added in Build Plan 00 §4
  **specifically** for this plan's PR-open flow; `starter_pack_id` is a live FK
  so "check for latest" always sees the pack's current `source_ref`.
- **GitHub App token minting** (Build Plan 00 §5 Part A/B): the accept/save/PR
  flow calls it; secret key names are Build Plan 00's. **Open (§11 #2):**
  whether the App backend mints directly or via a shared helper — the same
  question Build Plan 01 §10 left open for the Research Agent SP.
- **Research Agent Job re-trigger entry point** (Build Plan 01 Part F):
  **this is a gap, not a settled contract.** Build Plan 01's Job is
  `table_update`-triggered over `rss_silver` and enumerates new rows by
  watermark; it has **no** parameterized single-entry regeneration mode. Part D
  here needs one. The contract to add to Build Plan 01 (see §11 #2):
  a `jobs()`-invokable run mode `mode=regenerate_single` taking
  `research_log_id` + `override_label` + `installation_id`, which re-runs Build
  Plan 01 §5 Part F steps 12–18 for that one entry under the override label and
  writes a findings file Build Plan 03 re-ingests to the same `research_log_id`.
  **Must be reconciled with Build Plan 01 before Part D is built.**
- **Taxonomy buckets** (Build Plan 01 §5 Part F step 15): the
  `taxonomy_buckets` table here is exactly what Build Plan 01 reads to extend
  the `ai_classify` label set. Keep the read contract (per-installation
  `bucket_name` list) stable.
- **Data Access Gate → warehouse Analytics** and **`jobs()` fire-and-poll /
  120 s timeout**: locked in `PROJECT-PLAN.md` §2 Phase 3, not re-decided here.

## 7. Platform constraints to build against (to be cross-review-confirmed)

- **Apps reverse-proxy 120 s per-request timeout** — any App→Job call
  (Part D re-trigger) and any potentially-long GitHub operation must be
  fire-and-poll or backgrounded, never a synchronous wait. (`PROJECT-PLAN.md`
  §2 Phase 3; confirm against `databricks-apps`.)
- **Data access pattern gate** — warehouse Analytics (`useAnalyticsQuery`),
  not raw-`SELECT` custom endpoints; Lakebase synced table only as a measured
  fallback with the §1c caveats. (Confirm against `databricks-apps` /
  `databricks-apps-python`.)
- **Lakebase deploy-first schema ownership** — App SP must deploy before local
  dev against the review-queue tables or Postgres returns `42501` (§1c). Bake
  into the runbook and the §9 checklist.
- **Lakebase synced-table read grant** — if used, the App SP needs an explicit
  GRANT beyond `CAN_CONNECT_AND_CREATE` (§1c).
- **GitHub App consent is on github.com** — irrelevant to per-PR operations
  (those use the minted token), but the install/connect flow's consent screen
  is Build Plan 00's concern, not repeated here.
- **Installation-token scope** — the minted token must carry
  `contents:write` + `pull_requests:write` on the target repo (Build Plan 00
  §1a manifest permissions) for the accept/save PR flow to succeed; surface a
  clear error if a repo was connected without them.

## 8. Explicit fan-out map

**Within this plan (parallelizable once §4's schema and Part A's shell +
per-installation scoping are agreed):**
- Part A (shell / identity / scoping) is the **parent** — Parts B–G all assume
  it. Build it first, not in parallel.
- After Part A, these are largely independent and can fan out to separate
  implementers:
  - Part B (read path / Data Access Gate) + Part C (queue UI, diff, editor) —
    naturally one implementer since C consumes B, but B's warehouse wiring can
    be spiked in parallel.
  - Part E (accept→PR + frontmatter bump) — self-contained GitHub/token +
    frontmatter logic; can be built against a fixture `review_items` row before
    C is finished.
  - Part F (starter-pack check) — depends on Part E's PR path and the same
    diff renderer as Part C, so sequence it after E and C.
  - Part G (Skill Library browser) — shares Part E's PR/frontmatter path; can
    be built alongside F.
- **Part D (override re-trigger) is blocked** on the Build Plan 01 entry-point
  contract (§6, §11 #2) — do not start it until that is reconciled.

**Across plans:**
- **Blocked on Build Plan 00** (Lakebase schema/SP, GitHub App token) and
  **Build Plan 03** (`research_log` populated). Hard sequential.
- **Blocked on a Build Plan 01 change** for Part D only (the regeneration entry
  point). Parts A–C, E–G do not need it.
- **Feeds Build Plan 05** (Local Installer — consumes Part E's accept hook) and
  **Build Plan 06** (Phase 4 three-way conflict UI — renders inside Part G's
  browser). Define the extension points; don't build those here.

## 9. Validation checklist

- [ ] With `research_log` seeded with real rows, the queue lists one card per
      pending entry, scoped to the current `installation_id` only (no
      cross-installation leakage).
- [ ] Citations render as footnotes with working outbound links, read from the
      `research_log` VARIANT `citations` column (not the VS index).
- [ ] All three `proposed_diff` shapes render correctly: unified diff,
      new-file preview (null `target_skill_path`), and null/"classification
      only".
- [ ] Accept opens a real PR against the configured repo on a new branch,
      never merges, and records `pr_url` / `pr_state`.
- [ ] The opened PR's file carries correctly incremented frontmatter:
      `metadata.version` bumped, `updated_at`/`last_research_log_id` set,
      `base_cli_version`/`base_skills_repo_ref` untouched.
- [ ] Reject sets `decision = rejected` in Lakebase and writes **nothing** to
      `research_log`.
- [ ] Edit-then-accept commits the **edited** text, sets `edited = true`.
- [ ] Classification override to a new bucket inserts a `taxonomy_buckets` row
      and (once §11 #2 is resolved) fires the regeneration Job **fire-and-poll**,
      polling status into `classification_overrides` — verify no synchronous
      call exceeds the 120 s proxy timeout.
- [ ] After regeneration, the same-`research_log_id` row refreshes the card
      with a diff grounded in the override label.
- [ ] "Check for latest from `<starter pack>`" creates `origin =
      starter_pack_diff` review items in the same queue that accept→PR
      identically; verify it resolves the pack's **current** `source_ref` (live
      FK), not a stale snapshot.
- [ ] Skill Library browser opens and edits a skill with **no** pending
      proposal and saves it as a PR.
- [ ] Deploy-first order verified: attempting local dev before the App SP's
      first deploy fails with the documented `42501`, and the runbook says so.

## 10. Acceptance criteria (Definition of Done)

- The App deploys via the `-app` bundle (after `-infra` + `-ai-tools`), starts
  (explicit post-deploy start step), and is reachable.
- Review-queue Lakebase tables (§4) exist in the App SP's schema; all reads and
  writes are `installation_id`-scoped.
- `research_log` is read through the Data Access Gate (warehouse Analytics
  default), never a raw-SELECT custom endpoint.
- Accept / reject / edit-then-accept, classification override + regeneration
  re-trigger (fire-and-poll), accept→PR with correct frontmatter, starter-pack
  check into the same queue, and the view/edit-any-file browser all work per §9.
- No review decision is ever written back into `research_log`; no PR is ever
  merged by the App.
- The Local Installer hook (Part E step 16) and the three-way conflict UI
  extension point (Part G step 21) exist as defined seams, deferred to Build
  Plans 05 / 06 respectively.
- Every §11 open item is either resolved and recorded here, or explicitly
  carried forward with its downstream owner.

## 11. Open items carried forward (decisions to confirm in cross-review)

1. **Review-queue Lakebase data model (§4).** The `review_items` /
   `classification_overrides` / `taxonomy_buckets` shapes are this plan's
   proposal — confirm columns/enums, and specifically how proposed content is
   held for review (`proposed_content_ref`: read verbatim from `research_log`
   for research-origin items vs. dedicated storage for pack-diff items).
2. **The regeneration entry point into Build Plan 01's Research Agent Job
   (§5 Part D, §6).** Biggest cross-plan gap in this plan. Build Plan 01's Job
   has no single-entry-by-`research_log_id`-under-override-label run mode.
   Decide: add a parameterized `mode=regenerate_single` run mode to that Job
   (preferred — reuses the same synthesis/classify/extract logic) vs. a
   separate lightweight regeneration job owned here. **Must be reconciled with
   Build Plan 01 before Part D is built**, and Build Plan 01 §10's
   "SP mints GitHub token directly vs. calls back into the App" question is
   entangled with it.
3. **Where the §5 frontmatter version-bump logic lives (§5 Part E step 15).**
   Proposed: the App backend owns it, applied identically on accept and
   editor-save. Confirm the exact `metadata.version` increment rule
   (`+au.N` segment) and that `base_*` fields are never touched.
4. **Single queue holding two origins (§4, §5 Part F).** The
   `origin` discriminator + nullable `research_log_id` is this plan's approach;
   confirm it (vs. a separate pack-diff table unioned at read time).
5. **SQL warehouse for `research_log` reads (§3, §5 Part B).** New declared
   warehouse vs. referenced existing one, and which bundle owns/references it.
6. **One App serving all installations vs. one App per installation**
   (inherited from Build Plan 01 §10). This plan writes all data access
   `installation_id`-parameterized so either works, but the deployment model
   and the identity/authorization story differ and must be decided.
7. **Lakebase migration mechanism** — inherited unresolved from Build Plan 00
   §10 and Build Plan 01 §10; the review-queue tables here must use whatever
   gets decided there, and it drives §3's "which bundle owns the review-queue
   DDL" call.
8. **Reviewer authorization model.** Who may accept/reject/override (any
   workspace user with App access vs. a designated reviewer role), and the
   identity gate that must keep the deferred maintainer-only upstream-PR path
   (`PROJECT-PLAN.md` §6 open question #4) invisible to non-maintainers. Not
   built here, but the gate's existence must be designed for.

## 12. Changelog

- **Initial draft** (pre-cross-review): drafted from `PROJECT-PLAN.md` §2
  Phase 3, §1a, §1c, §1d, §5, and the frozen contracts in Build Plan 00 §4
  (Lakebase `installations`/`starter_packs`, GitHub App token minting) and
  Build Plan 03 §4 (`research_log` schema + stable `research_log_id`). Made
  explicit flagged decisions for the eight open items in §11 — most notably
  surfacing the **classification-override regeneration entry point** as a
  genuine cross-plan gap requiring a new parameterized run mode on Build Plan
  01's Research Agent Job (§5 Part D / §6 / §11 #2), the review-queue Lakebase
  data model with a two-origin `review_items` table (§4), and App-backend
  ownership of the §5 frontmatter version bump on both accept and editor-save
  (§5 Part E step 15).
- **Cross-review fold-in (`codex` mechanics + `pi` structure).**
  - **`pi` BLOCKING B1 — wrong `PROJECT-PLAN.md` section cited (2 sites).** The
    maintainer-only upstream-PR path was cited as `PROJECT-PLAN.md §4`
    ("Suggested build order"); the actual content is `§6` open question #4.
    Corrected both occurrences (§1 out-of-scope and §11 #8), and added an
    inbound `§11 #8` pointer from the §1 mention (resolving `pi`'s NIT that
    §11 #8 had no inbound reference). `pi` otherwise found **no numbering
    off-by-N bug** (all 21 steps across Parts A–G sequential) and verified every
    cross-file citation (BP00 §4, BP01 Part F/§10, BP03 §4, and the §5
    frontmatter / §1a / §1c / §1d claims) resolves.
  - **`codex` NIT — synced-table "not a DAB resource" is version-sensitive.**
    The `databricks-lakebase` skill itself notes DAB support for autoscaling
    synced tables is "not yet available," so §5 Part B step 5 now frames "not a
    DAB resource" as a build-time live-doc recheck, not a timeless fact. `codex`
    found **no BLOCKING mechanics errors** and confirmed the Data Access Gate,
    `jobs()` fire-and-poll + 120 s timeout, GitHub App token-mint model, AppKit
    lock-in, `research_log` VARIANT reads, and the real BP01 regeneration gap
    (§11 #2) are all mechanically sound.
