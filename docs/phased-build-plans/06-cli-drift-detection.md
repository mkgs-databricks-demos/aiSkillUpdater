# Build Plan 06 — Phase 4: Version Tracking + CLI-Drift Detection

**Source of truth:** `docs/PROJECT-PLAN.md` §2 Phase 4 (version tracking +
the three-way CLI-drift comparison), §5 (the versioning frontmatter this phase
reads as authoritative — `base_cli_version`, `base_skills_repo_ref`,
`metadata.version`), §1c (the version-tracking table is a Lakebase *projection*;
the repo frontmatter is the source of record), §1d (the CLI-drift Job lives in
`-ai-tools`; the version-tracking Lakebase table is `-infra`-side state), §6
open question #3 (the resolved CLI-source facts about how skills + the
compat mapping are fetched). This plan restates what's needed to build Phase 4;
it does not re-litigate design decisions already locked there — if something
here seems to conflict with `PROJECT-PLAN.md`, that file wins and this plan is
stale.

**Cross-reviewed:** _pending_ — a first draft. An independent structural pass
(`pi`) and an independent Databricks/mechanics fact-check pass (`codex`) will
review this draft; every BLOCKING finding from both will be folded into a
revised version, with a changelog at the bottom.

**What this phase is, in one sentence:** a Job that, whenever a new Databricks
CLI (`aitools`) version ships, resolves what skill content that CLI version
would install, compares it against the installation's own tracked skills using
the shared base each skill started from, and classifies every managed skill
into one of three drift outcomes — surfacing genuine three-way conflicts to the
human in the Review App rather than silently picking a winner.

**Why this phase exists (design rationale, not re-litigated):** the whole
project is premised on the CLI's bundled skills going stale, so a user's
project-maintained skill drifts *ahead* of the CLI. But the CLI also keeps
shipping new versions — sometimes catching up to, sometimes independently
diverging from, what this project already changed. Without a drift check, a user
never learns when the CLI has caught up (so they could re-base) or when the two
have diverged and need reconciliation. See `PROJECT-PLAN.md` §2 Phase 4.

> **Decisions made in this plan that `PROJECT-PLAN.md` does not pin down.**
> Phase 4 specifies the *what* (resolve CLI version → skills-repo ref via
> `cli-compat.json`, fetch that ref's `manifest.json` + files, compare against
> the tracking table's `base_cli_version` + `updated_version`, three outcomes)
> but leaves several *hows* open. This plan makes explicit, flagged choices for:
> **which repo `cli-compat.json` actually lives in** — `databricks/cli`
> (`internal/build/`), *not* `databricks-agent-skills` as Phase 4's prose
> implies; a real correctness nuance (§5 Part A, §7, §11 #1); the **exact
> three-way comparison algorithm** — how "newer / diverged / changed since base"
> is computed (§5 Part C, §11 #2); the **new-CLI-release trigger + cadence**,
> since no push exists (§5 Part D, §11 #3); the **version-tracking Lakebase
> table's DDL/bundle ownership + migration mechanism** (inherited from Build
> Plans 00/04 — §4, §11 #4); and the **re-base action's ownership seam** between
> this phase (detect) and Build Plan 04 (render + execute via its PR flow)
> (§6, §11 #5). Each discretionary choice is consolidated in §11 for the repo
> owner and cross-review to confirm or override.

---

## 1. Objective

Detect and classify CLI drift for every managed skill, and hand genuine
conflicts to the human. When this plan is done:

1. A **version-tracking table in Lakebase** holds, per managed skill per
   installation, the shared base (`base_cli_version`, `base_skills_repo_ref`),
   the project's current `metadata.version`, and the latest computed drift
   status — a queryable **projection** of the repo's authoritative `SKILL.md`
   frontmatter (§1c / §5), always re-derivable, never the source of record.
2. On each new CLI release (detected by the trigger in §5 Part D), a Job
   **resolves the CLI version → skills-repo ref** via `cli-compat.json`,
   **fetches** that ref's `manifest.json` + skill files from the public
   `databricks-agent-skills` repo over plain HTTPS (the same
   `raw.githubusercontent.com` pattern the CLI itself uses — no CLI install
   needed), and **classifies** each managed skill into one of three outcomes:
   - **CLI caught up** — CLI's shipped content is newer/equal and the project
     hasn't diverged → surface "consider re-basing onto the CLI's version."
   - **Project ahead** — the project's `updated_version` is still ahead and the
     CLI hasn't moved since the shared base → informational, no action.
   - **Conflict** — both diverged independently since `base_cli_version` → flag
     for **human reconciliation via a three-way diff**, never auto-resolved.
3. **Conflicts (and re-base suggestions) surface in the Review App** through the
   extension point Build Plan 04 §5 Part G step 21 left in the Skill Library
   browser — this plan produces the drift data + the three-way content; Build
   Plan 04's UI renders it and reuses its accept/PR flow to execute a re-base or
   reconciliation.

**Out of scope (owned elsewhere):** the three-way diff *UI* and the accept/PR
flow that executes a re-base (Build Plan 04 owns the App shell + PR path — this
plan feeds it data); writing the `metadata.version` / `base_*` frontmatter
(Build Plan 04 §5 Part E step 15 on accept; Build Plan 05 copies it) — this plan
only **reads** it; installing anything locally (Build Plan 05).

## 2. Prerequisites

- **The §5 versioning frontmatter is in use** on the managed skills
  (`metadata.version`, `base_cli_version`, `base_skills_repo_ref`,
  `updated_at`, `last_research_log_id`). `PROJECT-PLAN.md` §4's build order
  places this phase *after* the version-tag scheme is in use (i.e. after Build
  Plan 04 is writing it on accept). Without frontmatter to read, there is
  nothing to compare.
- **Build Plan 00 built and deployed** — the Lakebase instance/schema (owned by
  the App SP) that this phase's version-tracking table lives in, and the
  `installations` table (drift is per-installation, scoped by `installation_id`;
  each installation tracks its own configured repo). The table's DDL/bundle
  ownership is an open item (§11 #4).
- **Build Plan 04 built** — this phase's conflict/re-base output renders through
  Build Plan 04 §5 Part G step 21's extension point; the re-base action reuses
  Build Plan 04's accept/PR flow.
- **Public network egress** to `raw.githubusercontent.com` (and wherever
  `cli-compat.json` actually lives — §11 #1) for the unauthenticated fetches.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1c / §1d)

| Resource | Bundle | Notes |
|---|---|---|
| CLI-drift-detection Job | `-ai-tools` | Per §1d's judgment call: it reads Vector-Search-adjacent + Lakebase state but doesn't touch the index, and depends on the `-infra`-side version-tracking Lakebase table, so it sits in `-ai-tools`. Revisit if it gains a dependency on something `-ai-tools` itself creates. |
| Version-tracking Lakebase table (§4) | *open — §11 #4* | §1c/§1d treat it as `-infra`-side Lakebase state, but it lives in the App SP's Lakebase schema (like Build Plan 04's review-queue tables). Whether its DDL ships in `-infra` or with the App, and the migration mechanism, is the same unresolved question as Build Plan 00 §10 / Build Plan 04 §11 #7. |
| Conflict/re-base UI | `-app` (Build Plan 04) | Not built here — this plan feeds Build Plan 04's existing extension point. |

**No Vector Search or AI Functions.** This phase is pure fetch + structured
comparison; it needs no embedding model, no `ai_*` function, and no index.

## 4. Data model (version-tracking Lakebase table)

Lives in the App SP's Lakebase schema (§1c routes the version-tracking table to
Lakebase as a queryable projection). The repo's `SKILL.md` frontmatter stays the
**authoritative source of record** (§5); this table is refreshed from it + the
CLI fetch on each run, and is always re-derivable.

> **Proposed (§11 #4/#6)** — shape below is this plan's design call, to confirm
> in cross-review.

### `skill_version_tracking`
One row per managed skill per installation.

| Column | Type | Notes |
|---|---|---|
| `installation_id` | FK → `installations` | Drift is per-installation (each tracks its own configured repo). |
| `skill_name` | text | The skill directory name, e.g. `databricks-pipelines`. |
| `base_cli_version` | text | From the repo `SKILL.md` frontmatter (§5) — the CLI version this skill started from. |
| `base_skills_repo_ref` | text | From frontmatter — the `databricks-agent-skills` ref `base_cli_version` resolved to (so no historical re-resolution needed). |
| `project_version` | text | The repo's current `metadata.version` (the project's own increment). |
| `last_checked_cli_version` | text, nullable | The newest CLI version this row was compared against. |
| `last_checked_skills_repo_ref` | text, nullable | The ref `last_checked_cli_version` resolved to. |
| `cli_shipped_version` | text, nullable | The `metadata.version` the CLI's fetched `SKILL.md` carried at that ref (the CLI's own value, for comparison — never written back to the repo). |
| `drift_status` | enum(`in_sync`, `cli_caught_up`, `project_ahead`, `conflict`, `not_shipped_by_cli`, `new_bucket`) | The computed outcome (§5 Part C). `not_shipped_by_cli` = a skill/New-Feature bucket the CLI doesn't have; `new_bucket` = a user taxonomy bucket with no CLI counterpart. |
| `last_checked_at` | timestamptz | |
| `updated_at` | timestamptz | |

PK: `(installation_id, skill_name)`. This is a projection, so a full recompute
from the repo frontmatter + a fresh CLI fetch must be able to rebuild every row.

## 5. Task breakdown

### Part A — Resolve CLI version → skills-repo ref (`-ai-tools` Job)
1. Given a target CLI version (from the trigger, §5 Part D), resolve it to the
   compatible `databricks-agent-skills` ref via **`cli-compat.json`**. **Note
   the real location (§11 #1):** this file is `go:embed`'d in the
   **`databricks/cli`** repo at `internal/build/cli-compat.json`, **not** in
   `databricks-agent-skills` — `PROJECT-PLAN.md` §2 Phase 4's phrasing ("that
   repo's `cli-compat.json`") is imprecise. Fetch it from the public
   `databricks/cli` repo source at the CLI version's tag/ref (verify
   fetchability + exact path at build — §7, §11 #1). If the mapping can't be
   fetched, fall back to the CLI's changelog (as Phase 4 notes) or fail the run
   cleanly with a clear message rather than guessing a ref.

### Part B — Fetch the CLI's shipped skill content (`-ai-tools` Job)
2. At the resolved ref, fetch the `databricks-agent-skills` **`manifest.json`**
   and the relevant skill files over plain HTTPS from
   `raw.githubusercontent.com/databricks/databricks-agent-skills/<ref>/...` —
   the same unauthenticated pattern the CLI itself uses (`PROJECT-PLAN.md` §6
   #3). No CLI install, no auth. Use conditional GET / caching where practical
   (§7) since re-runs at the same ref are common.
3. For each managed skill, read the CLI's shipped `SKILL.md` at that ref and
   capture its `metadata.version` (the CLI's own value) + content, for the
   comparison in Part C.

### Part C — Three-way comparison + classify (`-ai-tools` Job)
4. Read the **project side** from the configured repo's current `SKILL.md`
   frontmatter (authoritative — §5): `base_cli_version`, `base_skills_repo_ref`,
   `metadata.version` (= `project_version`).
5. Establish the **shared base**: the content/version at `base_skills_repo_ref`
   (the ref the skill started from). The three sides of the comparison are:
   **base** (at `base_skills_repo_ref`), **CLI-current** (at the new version's
   resolved ref), **project-current** (the repo now).
6. **Classify** per skill (§11 #2 pins the exact algorithm — below is the
   intended logic):
   - `project-current` == `base` (project hasn't diverged) AND `CLI-current` moved past `base` → **`cli_caught_up`** (suggest re-base).
   - `project-current` moved past `base` AND `CLI-current` == `base` (CLI hasn't moved) → **`project_ahead`** (informational).
   - both `project-current` and `CLI-current` moved past `base`, independently → **`conflict`** (three-way diff for human reconciliation).
   - neither moved → **`in_sync`**.
   - CLI has no counterpart (New Feature / user bucket) → **`not_shipped_by_cli`** / **`new_bucket`**.
   Define "moved past / diverged" concretely (§11 #2): a combination of
   `metadata.version` comparison **and** content-hash equality against the base
   is recommended — a version-string bump alone, or a content change alone,
   should both count as "moved," and equality requires both to match. Do **not**
   rely on `metadata.version` semver ordering alone, since the project's
   `+au.N` segment and the CLI's own scheme aren't directly comparable.
7. **Upsert** the result into `skill_version_tracking` (§4): set
   `last_checked_cli_version`, `last_checked_skills_repo_ref`,
   `cli_shipped_version`, `drift_status`, `last_checked_at`. Never write
   anything back into the repo `SKILL.md` — this phase is read-only against the
   source of record.

### Part D — Trigger + cadence (`-ai-tools` Job)
8. **Detect a new CLI release** — there is no push, so poll: on a schedule
   (§11 #3 — cadence TBD, e.g. daily/weekly), check the `databricks/cli` GitHub
   releases/tags for a version newer than `last_checked_cli_version`; also
   expose a **manual "check now"** trigger from the Review App (Build Plan 04,
   via its `jobs()` fire-and-poll pattern). Run Parts A–C when a newer version
   is found (or on manual demand).
9. Make the run **idempotent**: re-checking the same CLI version against an
   unchanged repo produces the same `drift_status` rows (no spurious churn).

### Part E — Hand conflicts/re-base suggestions to the Review App (contract, `-app` renders)
10. Expose the drift results so Build Plan 04 §5 Part G step 21's extension
    point can render them: for `conflict` rows, provide the **three-way content**
    (base / CLI-current / project-current) for the diff; for `cli_caught_up`
    rows, provide the re-base suggestion. The **re-base / reconciliation action
    itself reuses Build Plan 04's accept/PR flow** (§6, §11 #5) — this plan
    detects and supplies data; Build Plan 04 renders and executes.

## 6. Cross-plan contracts (shared config — do not re-decide here)

- **§5 frontmatter is authoritative and read-only here.** This plan reads
  `base_cli_version` / `base_skills_repo_ref` / `metadata.version`; Build Plan
  04 §5 Part E step 15 *writes/increments* it on accept, Build Plan 05 copies it
  locally. This phase never writes it (§5 Part C step 7).
- **Build Plan 04 §5 Part G step 21 extension point** is where this phase's
  conflict/re-base output renders. Contract (§5 Part E): this plan supplies, per
  flagged skill, `drift_status` + (for conflicts) the three content versions
  (base / CLI-current / project-current). Build Plan 04 owns the UI and the
  accept/PR execution of a re-base — including re-writing `base_cli_version` /
  `base_skills_repo_ref` on a re-base (that's a frontmatter write, so it belongs
  to Build Plan 04's flow, not here — §11 #5).
- **Build Plan 00 Lakebase** — the version-tracking table lives in the App SP's
  schema; `installations` scopes drift per installation. DDL/bundle ownership +
  migration mechanism inherited unresolved (§11 #4).
- **The `manifest.json` + `raw.githubusercontent.com` fetch pattern** and the
  `cli-compat.json` mapping are the CLI-source facts resolved in
  `PROJECT-PLAN.md` §6 #3 — consumed here, not re-investigated (except the
  `cli-compat.json` *location* nuance, §11 #1).

## 7. Platform constraints to build against (to be cross-review-confirmed)

- **`cli-compat.json` lives in `databricks/cli`, not `databricks-agent-skills`**
  (§5 Part A, §11 #1): it's a `go:embed` artifact at
  `internal/build/cli-compat.json`. Confirm it is fetchable from the public
  `databricks/cli` repo source at a release tag, and the exact path, before
  relying on the fetch (the runtime CLI uses the embedded copy; this plan needs
  the source-tree copy).
- **Unauthenticated public GitHub fetches** to `raw.githubusercontent.com` are
  subject to rate limits — use conditional GET / caching, and handle 403/429
  gracefully; the same consideration Build Plan 02's corpus scrape has.
- **Serverless compute is sufficient** — pure HTTPS fetch + string/hash
  comparison + Lakebase upsert; no AI Functions, no index.
- **No `metadata.version` semver assumption** — the project's `+au.N` segment
  and the CLI's own versioning aren't directly comparable; the comparison must
  use content-hash + base-relative "moved" semantics, not raw version ordering
  (§5 Part C, §11 #2).
- **Lakebase deploy-first** (§1c) — the App SP must own its schema before this
  table is queried; the Job's Lakebase access assumes the App has deployed.

## 8. Explicit fan-out map

**Within this plan:**
- **Part A (resolve ref via `cli-compat.json`)** and **Part B (fetch content)**
  are a natural pair — B needs A's ref — but both are self-contained fetch logic
  that can be spiked independently of the comparison.
- **Part C (comparison + classify)** is the core logic and depends on A/B's
  fetched content + the repo frontmatter; the exact algorithm (§11 #2) should be
  nailed down before C is implemented.
- **Part D (trigger + cadence)** is independent plumbing — can be built in
  parallel with A–C and wired at the end.
- **Part E (hand-off contract to Build Plan 04)** is a thin data contract; agree
  its shape early since Build Plan 04's Part G extension point consumes it.

**Across plans:**
- **Blocked on Build Plan 04** (it writes the §5 frontmatter this reads, and
  owns the conflict-UI extension point + re-base PR flow) and **Build Plan 00**
  (Lakebase). Hard sequential on the frontmatter being in use.
- **Consumes** Build Plan 04/05's frontmatter provenance; **feeds** Build Plan
  04's Skill Library browser.
- **Note**: the Job logic (fetch/compare/classify) is real code — when built,
  it's an `implement` task for a coding sub-agent → its own PR → cross-review →
  human merge.

## 9. Validation checklist

- [ ] Given a known CLI version, Part A resolves the correct
      `databricks-agent-skills` ref via `cli-compat.json` fetched from the
      **correct** repo/path (§11 #1), and fails cleanly if it can't.
- [ ] Part B fetches `manifest.json` + skill files at that ref over
      unauthenticated HTTPS with no CLI install, honoring conditional GET.
- [ ] All three outcomes are produced on constructed fixtures: a skill where the
      CLI caught up (`cli_caught_up`), one where the project is ahead
      (`project_ahead`), and one where both diverged (`conflict`).
- [ ] `in_sync`, `not_shipped_by_cli`, and `new_bucket` are produced for their
      respective cases.
- [ ] "Moved / diverged" is computed by content-hash + base comparison, not
      `metadata.version` ordering alone — verify a content change with no version
      bump still counts as moved, and vice versa (§11 #2).
- [ ] The Job never writes to the repo `SKILL.md`; the Lakebase table fully
      rebuilds from repo frontmatter + a fresh fetch (projection property).
- [ ] Re-running against the same CLI version + unchanged repo is idempotent (no
      status churn).
- [ ] A new-CLI-release poll detects a version newer than
      `last_checked_cli_version`; the manual "check now" trigger runs on demand.
- [ ] Conflict rows expose base / CLI-current / project-current content for
      Build Plan 04's three-way diff; a re-base suggestion routes into Build Plan
      04's accept/PR flow (not executed here).
- [ ] Drift is scoped per `installation_id` — no cross-installation leakage.

## 10. Acceptance criteria (Definition of Done)

- The CLI-drift Job deploys via `-ai-tools`, resolves the ref, fetches CLI
  content unauthenticated, computes the three-way comparison, and upserts
  `drift_status` into the `skill_version_tracking` Lakebase table, scoped per
  installation.
- All drift outcomes are produced correctly on fixtures; the comparison uses
  content-hash + base semantics (not version ordering); the run is idempotent
  and never mutates the repo frontmatter.
- New-release detection (poll) + manual "check now" both trigger the run.
- Conflict/re-base data is handed to Build Plan 04's Part G extension point in
  the agreed shape; the re-base action reuses Build Plan 04's PR flow.
- Every §11 open item is either resolved and recorded here, or explicitly
  carried forward with its downstream owner.

## 11. Open items carried forward (decisions to confirm in cross-review)

1. **Where `cli-compat.json` actually lives + its fetchability (§5 Part A,
   §7).** It is `go:embed`'d in `databricks/cli` (`internal/build/
   cli-compat.json`), **not** `databricks-agent-skills` as Phase 4's prose
   implies. Confirm the source-tree copy is fetchable from the public
   `databricks/cli` repo at a release tag, the exact path, and its JSON shape
   (CLI version → skills-repo ref) — the whole resolution step depends on it.
2. **The exact three-way comparison algorithm (§5 Part C).** How "moved /
   diverged / newer" is defined — recommended: content-hash equality against the
   base **plus** `metadata.version` change, not semver ordering of
   `metadata.version` (the `+au.N` segment isn't comparable to the CLI's
   scheme). Confirm and pin.
3. **New-CLI-release trigger + cadence (§5 Part D).** No push exists; poll
   `databricks/cli` releases/tags on a schedule (pick a cadence) plus a manual
   "check now" from the App. Confirm the cadence and the release-source API.
4. **Version-tracking table DDL/bundle ownership + migration mechanism (§3,
   §4).** Same unresolved question as Build Plan 00 §10 / Build Plan 04 §11 #7 —
   whether the table's DDL ships in `-infra` or with the App, and the Lakebase
   migration mechanism.
5. **Re-base action ownership seam (§5 Part E, §6).** This plan detects +
   supplies three-way data; Build Plan 04 renders + executes the re-base via its
   accept/PR flow, including re-writing `base_cli_version` /
   `base_skills_repo_ref` on re-base (a frontmatter write that belongs to Build
   Plan 04, not here). Confirm the seam and the exact re-base frontmatter update
   rule.
6. **Table shape / projection semantics (§4).** Confirm `skill_version_tracking`
   columns/enums and that a full recompute from repo frontmatter + a fresh fetch
   rebuilds every row (projection, not source of record).

## 12. Changelog

- **Initial draft** (pre-cross-review): drafted from `PROJECT-PLAN.md` §2
  Phase 4, §5, §1c, §1d, §6 #3, and the extension-point/frontmatter contracts in
  Build Plan 04 (§5 Part E step 15, §5 Part G step 21) and Build Plan 05
  (provenance copy). Made explicit flagged decisions for the six open items in
  §11 — most notably surfacing that **`cli-compat.json` lives in `databricks/cli`
  (`internal/build/`), not `databricks-agent-skills`** as Phase 4's prose
  implies (§5 Part A / §7 / §11 #1), and specifying a **content-hash + base**
  comparison rather than `metadata.version` semver ordering (§5 Part C / §11 #2),
  since the project's `+au.N` segment isn't comparable to the CLI's own scheme.
