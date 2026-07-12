# Build Plan 07 — Phase 7: Guardrails & Observability

**Source of truth:** `docs/PROJECT-PLAN.md` §2 Phase 7 (failure alerting to
Slack; the "every change is a PR / locally-reviewed apply, never an unattended
auto-merge or auto-overwrite" guardrail). Phase 7 in the source doc is
deliberately thin — its real content is the set of **observability/alerting
hooks that every earlier build plan explicitly deferred here** (each emitted a
signal and said "Build Plan 07 alerts on it"). This plan consolidates those.
If something here seems to conflict with `PROJECT-PLAN.md`, that file wins and
this plan is stale.

**Cross-reviewed:** _pending_ — a first draft. An independent structural pass
(`pi`) and an independent Databricks/mechanics fact-check pass (`codex`) will
review this draft; every BLOCKING finding from both will be folded into a
revised version, with a changelog at the bottom.

**What this phase is, in one sentence:** the cross-cutting layer that (a)
watches every failure/anomaly signal the other phases emit and alerts a human
(primarily Slack) so a bad run never silently drops an announcement, and (b)
codifies + verifies the system-wide guardrail that no skill change ever reaches
a user without a human-reviewed PR or an explicitly-accepted local apply.

**Why this phase is last and cross-cutting (design rationale):** it adds almost
no new source data — every prior plan was deliberately built to *emit* a signal
(`errors` arrays, `_rescued_data`, `last_error` columns, `regeneration_status`,
Job `on_failure` hooks, polling-freshness timestamps) and defer the *alerting*
to here, so the signals already exist to watch. Building it last means it can
consolidate the real, final signal set rather than guessing. See
`PROJECT-PLAN.md` §2 Phase 7.

> **Decisions made in this plan that `PROJECT-PLAN.md` does not pin down.**
> Phase 7 names *what* to guard against (Slack alerts on RSS/LLM/GitHub
> failures + the `ai_classify` no-skill-match case; the no-auto-merge guardrail)
> but leaves the *how* almost entirely open. This plan makes explicit, flagged
> choices for: the **Slack delivery mechanism** per signal class — Jobs
> `webhook_notifications` for job-level vs. a monitoring sweep for data-level
> signals (§5 Part A/B, §11 #1); the **data-quality monitoring sweep** that
> catches the non-job-failure signals (non-empty `errors`, `_rescued_data`,
> stale polling cursor, `docs_scrape_state.last_error`,
> `regeneration_status=failed`) (§5 Part B, §11 #2); how the **`ai_classify`
> no-skill-match / New-Feature-only case** is detected given `ai_classify` has
> no native confidence (§5 Part B, §11 #3); **alert routing / dedup / throttle**
> to avoid alert storms (§5 Part D, §11 #4); the **guardrail codification +
> verification** approach (documented invariant vs. runtime assertion vs. test)
> (§5 Part C, §11 #5); and **bundle placement + the Slack webhook secret** (§3,
> §11 #6). Each is consolidated in §11 for the repo owner and cross-review to
> confirm or override.

---

## 1. Objective

Make failures visible and keep the human gate inviolable. When this plan is done:

1. **Job-level failures alert to Slack** — the RSS polling Job, the Research
   Agent Job, the ingestion pipeline, and the CLI-drift Job all route
   `on_failure` to a Slack destination (the hook Build Plan 01 §7 already wired
   `email_notifications`/`webhook_notifications.on_failure` for).
2. **Data-level anomalies alert to Slack** — a monitoring sweep catches the
   signals that are *not* job failures: non-empty `errors` arrays in
   `research_log` (per-entry AI-Function failures), non-null `_rescued_data` in
   the ingestion bronze (off-schema findings files), stale `rss_polling_cursor`
   (polling stopped advancing), `docs_scrape_state.last_error` (corpus fetch
   failures), `installations.status = failed` (stuck onboarding), and
   `classification_overrides.regeneration_status = failed`.
3. **The `ai_classify` "no existing skill matched" case surfaces** — when an
   announcement classifies only as `"New Feature"` (or an empty/degenerate label
   set), a human is told, so a genuinely-new capability isn't quietly filed away.
4. **Alerts are routed, deduped, and throttled** so a persistent failure
   doesn't produce a storm (one alert per distinct problem, not per poll).
5. **The no-auto-merge / no-auto-overwrite guardrail is codified and verified**
   — a documented, testable invariant that every skill change reaches a user
   only via a human-reviewed PR (Build Plan 04) or an explicitly-accepted local
   apply (Build Plan 05), never an unattended write. Nothing in the system
   merges a PR or overwrites a local file without human acceptance.

**Out of scope (owned elsewhere):** *emitting* the signals (each prior plan
already does — this plan only watches/alerts); the accept/PR flow itself (Build
Plan 04) and the local apply gate (Build Plan 05) — this plan *verifies* those
guardrails hold, it does not reimplement them.

## 2. Prerequisites

- **All prior build plans (00–06) built**, since this plan watches the signals
  they emit. In practice it can be built incrementally as each emitter lands,
  but its acceptance criteria assume the full signal set exists. The specific
  emitted signals it depends on (all already specified in their plans):
  - Job `on_failure` notification hooks — Build Plan 01 §7 (RSS + Research
    Agent Jobs); the ingestion pipeline (Build Plan 03) and CLI-drift Job
    (Build Plan 06) need the same hooks added if not already.
  - `research_log.errors` VARIANT (Build Plan 01 §5 Part F step 19 / Build Plan
    03 §4) — populated, non-empty on per-entry AI failures.
  - `research_findings_bronze._rescued_data` (Build Plan 03 §4).
  - `rss_polling_cursor.last_checked_at` freshness (Build Plan 01 §4).
  - `docs_scrape_state.last_error` (Build Plan 02 §4).
  - `installations.status`/`last_error` (Build Plan 00 §4).
  - `classification_overrides.regeneration_status` (Build Plan 04 §4).
- **A Slack destination** — an incoming-webhook URL (or a Databricks
  notification destination) stored as a secret; the delivery mechanism is an
  open item (§11 #1/#6).
- **Lakebase + the App SP schema deployed** (§1c) if any alert-state/dedup table
  (§5 Part D) lands in Lakebase.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

| Resource | Bundle | Notes |
|---|---|---|
| Job-level `on_failure` notifications | each Job's own bundle | Attached to the Jobs that already live in `-infra` (RSS, Research Agent, drift-adjacent) and `-ai-tools` (CLI-drift). This plan ensures every Job has the hook; the hooks ship with their Jobs, not as a separate resource. |
| Data-quality monitoring sweep (§5 Part B) | `-infra` *(proposed)* | It reads across the Delta state (`research_log`, bronze) and the Lakebase state (cursor, scrape-state, installations, overrides), all `-infra`-side, so a scheduled sweep Job belongs in `-infra` (§11 #6). |
| Slack webhook secret / notification destination | `-infra` secret scope | The scope is `-infra`'s (Build Plan 00 §3); the value is written at deploy/runtime, not in bundle YAML (same pattern as the GitHub App secrets). |
| Alert-state / dedup table (if used, §5 Part D) | *open — §11 #4/#6* | Small Lakebase OLTP state (last-alerted fingerprint per problem) — same DDL/bundle-ownership question as the other Lakebase tables (Build Plan 00 §10). |
| Guardrail invariant + verification (§5 Part C) | cross-cutting | Not a deployable — a documented invariant plus a test/audit; see §5 Part C. |

## 4. Signals inventory & delivery model (the consolidated deferred set)

Every signal below was **emitted by an earlier plan that deferred alerting to
here.** This plan adds no new source data; it watches these and alerts.

| Signal | Source (emitter) | Class | Delivery |
|---|---|---|---|
| RSS polling Job failure | Build Plan 01 §7 | job-level | `on_failure` → Slack (§5 Part A) |
| Research Agent Job failure | Build Plan 01 §7 | job-level | `on_failure` → Slack |
| Ingestion pipeline failure | Build Plan 03 | job-level | `on_failure` → Slack (add hook) |
| CLI-drift Job failure | Build Plan 06 | job-level | `on_failure` → Slack (add hook) |
| Non-empty `research_log.errors` (per-entry AI failure) | Build Plan 01 §5 Part F step 19 / Build Plan 03 §4 | data-level | monitoring sweep (§5 Part B) |
| Non-null `_rescued_data` (off-schema findings file) | Build Plan 03 §4 | data-level | monitoring sweep |
| Stale `rss_polling_cursor.last_checked_at` (polling stopped) | Build Plan 01 §4 | data-level | monitoring sweep (freshness) |
| `docs_scrape_state.last_error` (corpus fetch/parse fail) | Build Plan 02 §4 | data-level | monitoring sweep |
| `installations.status = failed` (stuck onboarding) | Build Plan 00 §4 | data-level | monitoring sweep |
| `classification_overrides.regeneration_status = failed` | Build Plan 04 §4 | data-level | monitoring sweep |
| `ai_classify` "no existing skill matched" (New-Feature-only) | Build Plan 01 §5 Part F step 15 | data-level | monitoring sweep (§5 Part B, §11 #3) |
| GitHub API / token-scope failure on accept→PR | Build Plan 04 §5 Part E | app-level | surfaced in-App + optionally Slack (§5 Part A) |

**Delivery split (design):** *job-level* failures use the platform's own Jobs
`webhook_notifications.on_failure` → Slack (no custom code). *Data-level*
signals need a **scheduled monitoring sweep** (§5 Part B) that queries the state
and emits Slack alerts, since they aren't job crashes. *App-level* failures
(GitHub) surface in the App UI (Build Plan 04 already shows a clear error) and
optionally also Slack.

## 5. Task breakdown

### Part A — Job-level failure alerting (each Job's bundle)
1. Ensure **every** Job in the system has `on_failure` notifications routed to
   the Slack destination: RSS polling + Research Agent (Build Plan 01 §7 already
   set `email_notifications`/`webhook_notifications.on_failure` — point them at
   Slack), the ingestion pipeline (Build Plan 03), and the CLI-drift Job (Build
   Plan 06). Prefer `webhook_notifications` → a Slack incoming webhook (or a
   Databricks notification destination) over email as the primary channel, per
   Phase 7's "Slack" (§11 #1); keep email as an optional backup.
2. Confirm `max_retries` is set (Build Plan 01 §7 already does for its Jobs) so
   a transient failure retries before it alerts — the alert should fire on
   *terminal* failure, not the first transient blip.

### Part B — Data-quality monitoring sweep (`-infra`)
3. Build a **scheduled sweep** (a small Job, or Databricks SQL Alerts over
   queries — pick one, §11 #2) that scans the data-level signals in §4 and emits
   a Slack alert per new problem:
   - non-empty `research_log.errors`; non-null `_rescued_data` in bronze;
   - `rss_polling_cursor.last_checked_at` older than a freshness threshold
     (polling should advance ~daily — Build Plan 01 — so e.g. >48h stale is an
     alert);
   - `docs_scrape_state.last_error` set; `installations.status = failed`;
   - `classification_overrides.regeneration_status = failed`.
4. **`ai_classify` no-skill-match detection (§11 #3):** `ai_classify` returns no
   native confidence (Build Plan 01 §5 Part F step 15), so "low confidence for
   all skills" is operationalized as **"the entry's only label is `"New
   Feature"` (or the label set is empty/degenerate)"** — i.e. it matched no
   existing skill. Alert on those so a genuinely-new capability is surfaced for
   a human to consider as a new skill, rather than filed quietly. (This is a
   feature signal, not an error — route it as informational, not critical.)
5. Scope every sweep query by `installation_id` and run it on serverless; it is
   pure read + alert, no writes to the source state.

### Part C — Guardrail codification + verification (cross-cutting)
6. **Codify the invariant** explicitly: *no skill change reaches a user except
   via (a) a human-reviewed PR to the configured repo (Build Plan 04 accept →
   PR, human merges) or (b) an explicitly-accepted local apply (Build Plan 05,
   gated behind an accepted/merged item).* The App never merges a PR; the
   installer never applies an unaccepted item; no pipeline or Job ever writes to
   the configured repo or a local skill directory unattended.
7. **Verify it** (§11 #5): the recommended approach is a documented invariant
   plus a lightweight audit/test — e.g. assert that the only code path writing to
   the configured repo is Build Plan 04's PR-open (never a merge call), and that
   Build Plan 05's apply is unreachable without an accepted item. Prefer a test
   / code-audit checklist over a runtime assertion where possible, since the
   guardrail is architectural. Do **not** add any auto-merge or auto-apply path
   "for convenience" — that would violate the invariant this plan exists to
   protect.

### Part D — Alert routing, dedup, throttle (`-infra`)
8. **Dedup + throttle** so a persistent problem alerts once, not every sweep/
   poll (§11 #4): keep a small last-alerted fingerprint per distinct problem
   (e.g. per `(installation_id, signal_type, subject_id)`) — in Lakebase (§3) or
   an equivalent — and only re-alert on a new problem or after a clear/cooldown.
   A resolved problem should optionally emit a recovery note.
9. **Severity routing:** distinguish critical (a Job down, polling stalled — a
   dropped announcement) from informational (a New-Feature-only classification,
   a single per-entry `errors` row) so the critical ones are unmissable and the
   informational ones don't cause fatigue.

## 6. Cross-plan contracts (shared config — do not re-decide here)

- **Every signal in §4 is emitted by its source plan; this plan only watches.**
  It must not require changing those emitters except to point existing
  `on_failure` hooks at Slack and to add the same hook to Jobs that lack it
  (Build Plan 03 pipeline, Build Plan 06 Job) — a small, additive change to
  those plans' Jobs, not a redesign.
- **Build Plan 01 §7 already set the Job notification hooks** (`max_retries`,
  `email_notifications`/`webhook_notifications.on_failure`) — this plan is "the
  later plan those hooks were built for" (Build Plan 01 §7 says exactly this).
- **The guardrail (Part C) verifies Build Plan 04's + Build Plan 05's behavior**
  (accept→PR-never-merge; apply-only-accepted) — it does not reimplement them;
  it asserts they hold system-wide.
- **Slack secret / notification destination** uses the `-infra` secret scope
  (Build Plan 00 §3), written at runtime, same pattern as the GitHub App
  credentials.

## 7. Platform constraints to build against (to be cross-review-confirmed)

- **Jobs `webhook_notifications` vs. `email_notifications`** — confirm the
  supported Slack path (a Slack incoming-webhook via `webhook_notifications`, or
  a Databricks notification destination) and prefer it over email for the
  primary channel (§11 #1). Build Plan 01's `databricks-jobs`
  `references/notifications-monitoring.md` is the reference.
- **Databricks SQL Alerts vs. a sweep Job** for the data-level signals (§5 Part
  B, §11 #2) — SQL Alerts are the native "query + threshold + notify" mechanism
  and may cover most data-level signals without custom code; confirm they can
  target Slack and read the Delta + Lakebase state, else use a small scheduled
  Job. Don't assume one without checking.
- **Serverless is sufficient** for the sweep (pure read + notify).
- **No new AI Functions / Vector Search** — the `ai_classify` no-match signal is
  read from existing `research_log` labels, not a new classification call.
- **Alert fatigue is a real failure mode** — a monitoring layer that cries wolf
  gets muted, defeating its purpose; the dedup/throttle (§5 Part D) is not
  optional polish.

## 8. Explicit fan-out map

**Within this plan:**
- **Part A (job-level notifications)** is small, additive, and independent —
  wiring existing `on_failure` hooks to Slack across the Jobs. Can be done first
  and standalone.
- **Part B (monitoring sweep)** is the largest piece and depends on the signal
  emitters existing; independent of Part A's job-level wiring. The mechanism
  choice (SQL Alerts vs. sweep Job, §11 #2) should be settled before building it.
- **Part C (guardrail verification)** is cross-cutting audit/test work,
  independent of A/B — can be done in parallel.
- **Part D (dedup/throttle/routing)** wraps Part B's output; build after B's
  signal set is producing alerts.

**Across plans:**
- **Depends on 00–06** for the signals it watches, but can be built
  incrementally as emitters land. Adds only additive changes to those plans'
  Jobs (Slack hook), never a redesign.
- **Note**: the sweep Job / alert logic is real code — when built, it's an
  `implement` task for a coding sub-agent → its own PR → cross-review → human
  merge.

## 9. Validation checklist

- [ ] A forced terminal failure of each Job (RSS, Research Agent, ingestion
      pipeline, CLI-drift) produces a Slack alert via `on_failure`, and only
      after `max_retries` is exhausted (not on the first transient blip).
- [ ] A fixture `research_log` row with a non-empty `errors` array produces a
      data-level Slack alert from the sweep.
- [ ] A non-null `_rescued_data` bronze row produces an alert.
- [ ] A `rss_polling_cursor.last_checked_at` older than the freshness threshold
      produces a "polling stalled" alert.
- [ ] `docs_scrape_state.last_error`, `installations.status = failed`, and
      `classification_overrides.regeneration_status = failed` each produce an
      alert.
- [ ] An announcement classified **only** as `"New Feature"` produces an
      **informational** (not critical) surface for a human (§5 Part B step 4).
- [ ] Dedup/throttle: a persistent failure across N sweeps produces **one**
      alert, not N; a newly-distinct problem still alerts.
- [ ] Severity routing distinguishes critical (dropped announcement / Job down)
      from informational.
- [ ] Guardrail audit: there is **no** code path that merges a PR or writes a
      skill file (repo or local) without a human-accepted item — verified by the
      Part C audit/test.
- [ ] Every alert query is scoped per `installation_id` (no cross-installation
      leakage).

## 10. Acceptance criteria (Definition of Done)

- Every Job routes terminal `on_failure` to Slack (post-retry); the data-quality
  sweep alerts on all §4 data-level signals; the New-Feature-only case surfaces
  informationally.
- Alerts are deduped/throttled and severity-routed so critical failures are
  unmissable and informational ones don't cause fatigue.
- The no-auto-merge / no-auto-overwrite guardrail is codified as a documented
  invariant and verified by an audit/test; no unattended write path exists.
- The monitoring layer adds no new source data and only additive Job-notification
  changes to prior plans.
- Every §11 open item is either resolved and recorded here, or explicitly
  carried forward with its downstream owner.

## 11. Open items carried forward (decisions to confirm in cross-review)

1. **Slack delivery mechanism per signal class (§5 Part A/B, §3).** Jobs
   `webhook_notifications` (Slack incoming webhook) vs. a Databricks
   notification destination for job-level; and how data-level alerts reach the
   same Slack channel. Confirm the supported path and prefer it over email.
2. **Data-quality monitoring mechanism (§5 Part B).** Databricks SQL Alerts
   (native query+threshold+notify) vs. a small scheduled sweep Job — confirm SQL
   Alerts can target Slack and read both Delta and Lakebase state, else use a
   Job. Pick one.
3. **`ai_classify` no-skill-match detection (§5 Part B step 4).** Confirm the
   operational definition ("only label is `New Feature`, or empty/degenerate
   labels") given `ai_classify` has no native confidence (Build Plan 01 §5 Part
   F step 15), and that it routes as informational, not critical.
4. **Alert dedup/throttle + state (§5 Part D).** The fingerprint key
   (`installation_id, signal_type, subject_id`), the cooldown, recovery notes,
   and where the last-alerted state lives (Lakebase vs. alert-mechanism-native).
5. **Guardrail verification approach (§5 Part C).** Documented invariant +
   audit/test (preferred) vs. a runtime assertion — confirm how the
   "no unattended write / no auto-merge" invariant is actually enforced and
   tested.
6. **Bundle placement + Slack secret + any alert-state table (§3).** Sweep Job
   in `-infra`; Slack webhook value in the `-infra` secret scope; the
   alert-state table (if used) shares the unresolved Lakebase DDL/migration
   question (Build Plan 00 §10).

## 12. Changelog

- **Initial draft** (pre-cross-review): drafted from `PROJECT-PLAN.md` §2
  Phase 7 (thin — two bullets) by **consolidating the observability/alerting
  hooks every earlier plan deferred here**: Build Plan 00 (`status`/`last_error`),
  Build Plan 01 (Job `on_failure` hooks, `errors` arrays, polling freshness, the
  `ai_classify` no-match case), Build Plan 02 (`docs_scrape_state.last_error`),
  Build Plan 03 (`_rescued_data`), Build Plan 04 (`regeneration_status`, GitHub
  errors), and Build Plan 06 (drift-check failures). Framed the phase as two
  things — a watch/alert layer over already-emitted signals, and codification +
  verification of the system-wide no-auto-merge/no-auto-overwrite guardrail.
  Made explicit flagged decisions for the six open items in §11 — most notably
  the job-level-vs-data-level delivery split (§4), the SQL-Alerts-vs-sweep-Job
  mechanism (§11 #2), and operationalizing the `ai_classify` no-match signal
  without native confidence (§11 #3).
