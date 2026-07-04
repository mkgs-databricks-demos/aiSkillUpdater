# aiSkillUpdater — Project Plan

**Goal:** turn the current manual "run `/investigate` in Polly, review the
diff, copy files by hand" workflow (see `docs/STAGING-AREA.md`) into a
scheduled Databricks App that keeps the Databricks skill library current
automatically, with a human approval gate before anything ships.

## 0. Current state (baseline)

- This repo is the staging area: one directory per skill, each with an
  updated `SKILL.md` / `references/` and a `REVIEW-BEFORE-COPYING.md` diff
  summary.
- Audits are triggered manually and run by ad hoc sub-agents that cross-check
  each `SKILL.md` against live `docs.databricks.com` pages.
- Deploying an update means `cp -r` from this repo into
  `~/.databricks/aitools/skills/<name>/` on the machine running Claude Code.

Everything below replaces the "manual, on-demand" parts with "scheduled,
automatic, PR-gated" — the human review step stays, it just moves from
"review a folder" to "review a GitHub PR."

## 1. Target architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Databricks Job (serverless, scheduled)                          │
│                                                                   │
│  1. Refresh docs corpus  → UC Volume + Vector Search index        │
│  2. For each skill in this repo (via GitHub API):                │
│       - retrieve grounding context from the Vector Search index   │
│       - ask an LLM (Foundation Model API / ai_query) to diff the  │
│         skill against current docs and propose an update          │
│       - write result row to a Delta audit-log table              │
│       - if changed: open a branch + PR against                    │
│         mkgs-databricks-demos/aiSkillUpdater                      │
│  3. Post a Slack/PR summary                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Databricks App (AppKit + React, thin FastAPI/Node backend)       │
│  - Dashboard: recent audit runs, open PRs, last-updated per skill │
│  - "Run audit now" button (per skill or full sweep) → triggers    │
│    the Job via Jobs API                                           │
│  - Read-only — merging/deploying is still a human GitHub action   │
└─────────────────────────────────────────────────────────────────┘
```

A human always merges the PR (same rule Polly itself follows: sub-agents
never merge). Deployment to the live skill directory is a separate,
explicit step — see Phase 4.

## 2. Build phases

### Phase 1 — Grounding corpus (docs ingestion)
- Scrape/sync the relevant `docs.databricks.com` sections (one job per
  product area: SQL, Apps, Jobs, ML, UC, etc.) into a UC Volume.
- Chunk + embed into a Vector Search index (reuses the
  `databricks-vector-search` skill's index-types/end-to-end-rag patterns
  already staged in this repo).
- Refresh on the same schedule as the audit job (or more often) so the
  grounding is never far behind live docs.

### Phase 2 — Audit engine (the Job)
- One serverless Databricks Job task, parameterized per skill or run as a
  fan-out over all skills (`databricks-jobs` skill patterns apply).
- Per skill: pull `SKILL.md` + `references/**` from this GitHub repo, run a
  RAG-grounded prompt ("here is the skill text, here is current docs
  context, list outdated claims + proposed replacement text") via `ai_query`
  or a Model Serving endpoint fronting Claude.
- Write one row per skill per run to a Delta table (`audit_log`): skill
  name, run timestamp, changed (bool), summary, PR URL.
- If the model proposes a real change: create a branch
  (`audit/<skill>-<date>`), commit updated files + a regenerated
  `REVIEW-BEFORE-COPYING.md`, open a PR via the GitHub REST API (token in a
  Databricks secret scope). Never push to `main` directly.

### Phase 3 — Review dashboard (the App)
- Databricks App (`databricks-apps` / `databricks-apps-python` skill
  patterns) reading the `audit_log` Delta table.
- Table view: skill, priority (P0/P1/P2 — reuse the existing inventory
  table), last audited, last changed, open PR link.
- Button to trigger an ad hoc run for one skill (calls the Jobs API
  `run_job` with a `skill_name` job parameter) — covers the "new skill
  installed" and "wrong code in the field" triggers from the cadence table
  without waiting for the schedule.

### Phase 4 — Deployment path
- PRs are merged by a human on GitHub, same as any other change.
- Add a small `sync` script (or a second, manually-triggered Job) that,
  given a merged commit SHA, copies the changed skill directories from a
  fresh clone into `~/.databricks/aitools/skills/` — replacing today's
  manual `cp -r` loop in `docs/STAGING-AREA.md`.
- If these skills are actually distributed as a Claude Code plugin
  (marketplace-published), prefer bumping the plugin version and going
  through the marketplace PR flow instead of a raw file copy — check
  whether `~/.databricks/aitools/skills/` is fed by a plugin install before
  building the file-copy path.

### Phase 5 — Guardrails & observability
- Schedule: weekly, **plus** an ad hoc trigger on every new DBR major
  release (reuses the existing cadence table in `docs/STAGING-AREA.md`).
- Failure alerting to Slack (job failure, LLM call failure, GitHub API
  failure) so a bad run doesn't silently stop audits.
- Every proposed change ships as a PR, never an auto-merge — same
  human-in-the-loop bar as the rest of this workflow.

## 3. Skills this reuses

The staging area already contains audited skill content for every
Databricks building block this project needs — treat these as the
reference material while implementing each phase:

| Phase | Relevant staged skill |
|-------|------------------------|
| Docs corpus + retrieval | `databricks-vector-search`, `databricks-unity-catalog` (volumes) |
| Audit engine | `databricks-jobs`, `databricks-ai-functions`, `databricks-dabs` |
| Review dashboard | `databricks-apps`, `databricks-apps-python`, `databricks-app-design` |
| Bundling everything | `databricks-dabs` (ship Job + App as one bundle) |

## 4. Suggested build order

1. Phase 2 first, run manually (no schedule) against one skill end-to-end,
   to prove the audit-and-PR loop works before investing in retrieval
   quality.
2. Phase 1 (real grounding corpus) once Phase 2's prompt/loop is proven —
   swap the "no grounding" placeholder for the Vector Search index.
3. Phase 3 (dashboard) once there's enough audit history to be worth
   viewing.
4. Phase 5 (schedule + alerting) last, once a human has watched a few
   manual runs and trusts the output enough to let it run unattended.
5. Phase 4 (deployment sync) can happen any time after Phase 2 — it's
   independent of the others.

## 5. Open questions for the repo owner

- Where does `~/.databricks/aitools/skills/` actually get populated from
  today — a plugin install, a manual clone, something else? This decides
  whether Phase 4 is a file copy or a plugin-marketplace version bump.
- Which LLM should the audit engine call — a Databricks-hosted external
  model pointing at Claude, or a Foundation Model API model? Affects the
  `ai_query` vs. Model Serving endpoint choice in Phase 2.
- Does "keep up to date" include auto-detecting *new* Databricks features
  that should become entirely new skills, or only refreshing existing
  skills? The current cadence table only covers the latter.
