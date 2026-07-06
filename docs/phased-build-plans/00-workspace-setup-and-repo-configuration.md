# Build Plan 00 — Phase 0: Workspace Setup / Repo Configuration

**Source of truth:** `docs/PROJECT-PLAN.md` §1a, §1c, §1d, §3, §4, §5, Phase 0
section. This plan restates what's needed to build Phase 0; it does not
re-litigate design decisions already locked there — if something here
seems to conflict with `PROJECT-PLAN.md`, that file wins and this plan is
stale.

**Cross-reviewed:** an independent structural pass (`pi`) and an
independent Databricks/GitHub-mechanics fact-check pass (`codex`) both
reviewed a first draft of this plan; every BLOCKING finding from both is
folded into the version below. See the bottom of this file for a summary
of what changed.

**Why this phase is first:** every other phase either reads/writes the
configured skills repo (Phase 3, Phase 5) or reads Phase 0's Lakebase
config (cloud/region selection feeds Phase 2's RSS ingestion). Per
`PROJECT-PLAN.md` §4 build-order note, Phase 0's plumbing must be exercised
by this project's *own* development too — there is no hardcoded
"this repo" path anywhere downstream.

---

## 1. Objective

Stand up the one-time, per-deployment GitHub App registration and the
per-installation onboarding flow that lets a Databricks App:

1. Connect to a user-chosen GitHub repo (the "configured skills repo" for
   that installation) with no manually-handled PAT or private key.
2. Let the user choose a starting point (empty repo vs. seed from a named
   starter pack).
3. Capture which cloud(s) and region(s) this installation tracks.
4. Persist all of the above as per-installation OLTP state in Lakebase.
5. Deploy the `-infra` → `-app` bundle sequence for the first time,
   proving the deploy-order/start-the-App plumbing end-to-end. **The
   `-ai-tools` bundle does not exist yet in this phase** (it's introduced
   in Build Plans 01/03) — Phase 0 only proves two of the eventual
   three bundles' ordering; see §3 and §9 item 1.

Everything in this phase is plumbing — no RSS ingestion, no research
agent, no review UI content yet. Done when a fresh installation can
authenticate to GitHub, pick a repo + starting point + cloud/region, and
land that config queryable in Lakebase, with the App itself reachable at
its public host.

## 2. Prerequisites

- None from other build plans — this is the first phase built.
- External: a GitHub organization/account where the one-time GitHub App
  registration will live. **This is a repo-owner decision, not an
  engineering task — confirm it before any Part A work starts** (see the
  "must be sequential" note in §6).
- Databricks workspace with Unity Catalog enabled, permission to create
  catalogs/schemas, Lakebase enabled for the workspace/region, and
  Databricks Apps enabled.
- Databricks CLI installed and authenticated (a valid profile / OAuth
  session) with bundle-deploy permissions in the target workspace — the
  implementer will run `databricks bundle deploy` and `databricks bundle
  run` (or equivalent App start command) as part of §5 Part C.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

| Resource | Bundle | Notes |
|---|---|---|
| UC catalog/schema (shared foundation for all phases) | `-infra` | Created once; every later phase's tables/volumes live under it |
| Secret scope for GitHub App credentials | `-infra` | Bundle declares the **scope only**; credential *values* are written at runtime by the App backend (§1a), never in bundle YAML |
| Secret-scope ACL grant (App service principal → `CAN WRITE`/`MANAGE`) | `-infra` | Platform default is READ-only — this must be an explicit grant, either declared in the bundle if the DAB schema supports ACL resources, or a documented one-time `databricks secrets put-acl` command run as part of the deploy runbook. Don't skip this — the manifest callback (§5 Part A) fails without it. |
| Lakebase database instance (+ branch, if used for dev/staging) | `-infra` | Backs all of §1c's OLTP tables, not just Phase 0's |
| Service principal — the App's own | `-infra` | Created here so grants in later bundles can target it |
| Databricks App resource + its `app.yaml` runtime config | `-app` | Deployed **last** of the two bundles that exist in this phase — see §7 for the explicit post-deploy start step. `app.yaml` (not `databricks.yml`) is where the App's runtime env vars go: secret-scope name, Lakebase connection details, GitHub App's public client ID (non-secret). Mutable per-installation settings (repo, cloud, region) never go in either bundle file — they live in Lakebase, written after deploy. |

`-ai-tools` does not exist in this phase at all — nothing in Phase 0
depends on Vector Search or Genie, and attempting to deploy it before
Build Plan 01/03 land is expected to fail by design (see §9 item 1).
Confirm the `-infra` bundle's schema/volume declarations are scoped
generically enough that Phase 1/Phase 2/Phase 2b can add their own
tables/volumes to the *same* `-infra` bundle later without restructuring
it — but don't over-engineer this "generic enough" call; a plain UC
schema + secret scope + Lakebase instance + two service principals is
already generic, no premature abstraction needed.

## 4. Data model (Lakebase)

Create these tables in the App's own Postgres schema (owned by the App's
service principal on first deploy — see the deploy-first ownership
pitfall in §7). The schema/migration mechanism is an implementation
choice (raw SQL migration files, Alembic, or the chosen framework's own
migration tool) — whichever is picked, **record it here once decided**
(see §10), because Build Plans 01 (RSS cursor table), 04 (review-queue
state), and 06 (version-tracking projection) add tables to this same
schema and must reuse the same migration mechanism rather than each
picking their own.

### `installations`
One row per Databricks workspace installation (there is exactly one row
in a self-hosted deployment, but design for multi-row from day one since
distributability is a hard requirement — see `PROJECT-PLAN.md` intro).

Onboarding is a multi-step flow with real external side effects (GitHub
redirects, token minting, a starter-pack commit) that cannot be rolled
back atomically the way a single-database transaction can. Model it as a
**durable state machine** keyed by `status`, not as one big all-or-nothing
transaction — a Lakebase transaction is only used for atomically updating
this one row at each step, never across the GitHub API calls in between.

| Column | Type | Notes |
|---|---|---|
| `installation_id` | uuid, PK | This project's own row identifier — **not** the GitHub App installation ID (that's a separate column below) |
| `status` | enum(`pending`, `github_connected`, `starting_point_chosen`, `cloud_region_chosen`, `completed`, `failed`) | Drives resumability — see §5 step 12 |
| `github_installation_id` | text, UNIQUE, nullable until `status ≥ github_connected` | The ID GitHub returns from the App-installation callback |
| `github_account_login` | text, nullable | The org/user login GitHub reports the app was installed on — useful for the settings screen and audit |
| `github_repo_owner` | text, nullable | Resolved via the GitHub API (see §5 step 7a) — not assumed to be directly present in the callback params |
| `github_repo_name` | text, nullable | Same as above |
| `default_branch` | text, nullable | Resolved alongside repo owner/name; needed later by Build Plan 04's PR-open flow |
| `starting_point` | enum(`scratch`, `starter_pack`), nullable | |
| `starter_pack_id` | FK → `starter_packs.pack_id`, nullable | Set only when `starting_point = starter_pack`. Because this is a live FK (not a copied snapshot), `PROJECT-PLAN.md` §1a's "starter packs stay live reference sources" requirement is satisfied automatically — Build Plan 04's "check for latest from Matt's version" always resolves this installation's pack row for current `source_repo`/`source_ref`, no separate per-installation snapshot table needed. |
| `last_error` | text, nullable | Set when `status = failed`; cleared on successful retry |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |
| `completed_at` | timestamptz, nullable | Set when `status = completed` |

Uniqueness: `github_installation_id` is UNIQUE when set (prevents the same
GitHub App installation from being claimed by two rows). Two different
installations of *this* GitHub App pointed at the *same* underlying repo
are allowed (not blocked) — GitHub itself is the source of truth for
whether that's sensible; this project doesn't add an extra constraint on
top.

### `tracked_clouds`
| Column | Type | Notes |
|---|---|---|
| `installation_id` | FK | |
| `cloud` | enum(`aws`, `gcp`, `azure`) | |

PK: `(installation_id, cloud)` — one row per selected cloud, multi-select
via multiple rows, duplicate selection rejected at the DB layer.

### `tracked_regions`
| Column | Type | Notes |
|---|---|---|
| `installation_id` | FK | |
| `cloud` | enum(`aws`, `gcp`, `azure`) | A region string alone is ambiguous across clouds (e.g. region-naming collisions) — always paired with which cloud it belongs to |
| `region` | text | e.g. `us-west-2` |

PK: `(installation_id, cloud, region)`. The UI should only offer regions
for clouds already present in this installation's `tracked_clouds`.

### `starter_packs` (registry — seeded with one row at build time)
| Column | Type | Notes |
|---|---|---|
| `pack_id` | text/uuid, PK | Stable identifier — not the display name, so a future rename doesn't break `installations.starter_pack_id` FKs |
| `pack_name` | text, UNIQUE | Display text, e.g. `"Matt's version"` |
| `source_repo_owner` | text | |
| `source_repo_name` | text | |
| `source_ref` | text | e.g. `main` or a tag |

This table is the generic registry `PROJECT-PLAN.md` §1a calls for — adding
a second starter pack later is a row insert, not new code. Because
`installations.starter_pack_id` is a live FK rather than a copied value,
updating a pack's `source_ref` here automatically becomes what every
linked installation's "check for latest" diffs against next time (Session
Plan 04) — no migration needed when a pack's reference moves.

### GitHub App credentials — **do not store here**
The app-level GitHub App credentials are **secrets**, not Lakebase rows —
they go in the Databricks secret scope declared by `-infra` (§3), written
at runtime per §5 Part A. Use these stable secret key names so later
plans/ops scripts can rely on them without re-deriving:

- `github-app-id`
- `github-app-private-key-pem`
- `github-app-webhook-secret`
- `github-app-client-id`
- `github-app-client-secret`

Writing these secrets must be **idempotent and overwrite-safe**: before
writing, check whether `github-app-id` already exists; if so, the
"register GitHub App" screen should show an "already registered" state
(showing the existing app ID/name, not the secret values) rather than
silently re-registering a second GitHub App and orphaning the first.

## 5. Task breakdown

**Execution-order note before starting:** Part A's manifest (steps 1–5)
cannot be *finalized* until Part C's first `-app` deploy (steps 15–16)
has assigned this installation's public `*.databricksapps.com` host —
GitHub needs that host as the manifest's callback URL. Build Part A's
form/callback *code* in parallel with Part C's bundle YAML, but treat
"submit the real manifest to GitHub" as blocked on Part C step 16
completing first. See §6 for how this is reflected in the fan-out map.

### Part A — One-time GitHub App registration (app-level, not per-installation)
1. Build an admin-only setup screen in the App that assembles a GitHub App
   **Manifest** JSON with the fields GitHub requires: `name`, `url`
   (marketing/info URL), `hook_attributes.url` (webhook URL) and
   `hook_attributes.active` (set `false` initially if webhooks aren't
   used yet — see step 1a), `redirect_url` (this app's manifest-callback
   endpoint), `callback_urls` (OAuth callback, if user-to-server auth is
   also needed — otherwise omit), `public` (`false`, this app is only
   ever installed by this project's own installations), `default_permissions`
   (`contents: write`, `pull_requests: write`, `metadata: read` — GitHub
   requires `metadata: read` implicitly for repo discovery), and
   `default_events` (empty array if webhooks aren't consumed yet).
   - **1a.** Decide now whether this phase actually needs webhook events
     (e.g. to react to PR-merged events later). If not needed yet, set
     `hook_attributes.active: false` and an empty `default_events` — this
     is a valid, minimal manifest and avoids building unused webhook
     receiver code in this phase. Note the choice in §10.
2. Implement the manifest submission per GitHub's actual documented flow:
   render an auto-submitting HTML form whose `action` is
   `https://github.com/settings/apps/new?state=<random-state>` with the
   manifest JSON as a hidden `manifest` field, and redirect the admin's
   browser to submit it. GitHub shows its own confirm/name screen, then
   redirects to this app's `redirect_url` with a one-time `?code=...`
   query param (validate `state` matches what was sent).
3. Implement the callback endpoint that receives that `code` and
   exchanges it server-side via `POST
   https://api.github.com/app-manifests/{code}/conversions` (no auth
   header needed — the code itself is the credential) to receive: app ID,
   private key (PEM), webhook secret, client ID, client secret.
4. Write all five values to the secret scope declared by the `-infra`
   bundle (§3), using the key names listed in §4's "GitHub App
   credentials" note. Never write to Lakebase, never to local/ephemeral
   disk (App filesystem is ephemeral — see §7). Guard against
   re-registration per §4's idempotency note.
5. Add a "GitHub App registered" status indicator to the admin screen
   (reads `github-app-id` from the secret scope's metadata/existence, not
   its value) so this doesn't need to be re-run per installation.

### Part B — Per-installation onboarding flow
6. Build a "Connect GitHub repo" button that redirects to GitHub's **App
   installation picker** (`https://github.com/apps/<app-slug>/installations/new`),
   scoped to org/repo selection. Create the `installations` row at
   `status = pending` before redirecting, so a partial/abandoned flow is
   traceable.
7. Implement the installation callback endpoint: receive
   `installation_id` and `setup_action` from GitHub's redirect query
   params.
   - **7a.** GitHub's callback does not reliably hand back the selected
     repository's owner/name directly — use the freshly-installed
     `github_installation_id` to mint an installation access token (see
     step 8, do this step's minting first if needed) and call GitHub's
     "list repositories accessible to the installation" API to resolve
     the actual `github_repo_owner`/`github_repo_name`/`default_branch`.
     If the installation has access to more than one repo, prompt the
     user to pick which one is *this installation's* configured skills
     repo (the GitHub App install and "which repo is the skills repo"
     are logically separate choices, even though the common case is one
     repo per install).
   - Update the `installations` row: `github_installation_id`,
     `github_account_login`, `github_repo_owner`, `github_repo_name`,
     `default_branch`, `status = github_connected`.
8. Implement server-side installation-access-token minting (short-lived,
   on demand, cached in memory only for the duration of a request — never
   persisted) using the app-level credentials from Part A + this
   installation's `github_installation_id`. No PAT anywhere in this flow.
9. Build the "starting point" step: scratch vs. starter pack, listing
   `starter_packs` rows. If starter pack chosen, copy that pack's
   `source_repo`@`source_ref` content into the newly-connected repo as
   its first commit (implementation detail: either a server-side git
   clone+push using the freshly-minted installation token, or an initial
   PR — pick whichever is simpler given the chosen App framework, note
   the choice in §10 since Build Plan 04 should be consistent about
   whether *all* repo writes go through PRs or this first-commit case is
   a deliberate exception). If "scratch" is chosen, take no repo action
   at all — the repo stays exactly as GitHub created it. Update
   `installations.starting_point` (+ `starter_pack_id` if applicable),
   `status = starting_point_chosen`.
10. Build the cloud-selection step: multi-select AWS/GCP/Azure, default
    pre-checked = this installation's auto-detected deployment cloud via
    `WorkspaceClient().config.cloud` (confirmed working in a Databricks
    App's default runtime credentials per `PROJECT-PLAN.md` §6 #10). If
    that call returns null/unknown for any reason, fall back to no
    pre-checked default rather than failing the onboarding step — cloud
    selection remains a plain, always-available multi-select either way.
    Write selections to `tracked_clouds`.
11. Build the region-selection step: multi-select from a **validated
    per-cloud region list** (not arbitrary free text) scoped to whichever
    clouds were chosen in step 10, with **no pre-filled default** (per
    `PROJECT-PLAN.md` §6 #12 — no reliable auto-detection exists;
    best-effort account-API check only if those credentials happen to be
    configured, otherwise blank). Write selections to `tracked_regions`
    (with the paired `cloud` column).
12. Update `installations.status = cloud_region_chosen`, then
    `status = completed`, `completed_at = now()` once all of steps 7–11
    have succeeded. Each step's Lakebase write is its own small
    transaction; the overall flow's "did onboarding finish" question is
    answered by reading `status`, not by a single cross-step database
    transaction (external GitHub calls can't participate in a Postgres
    transaction anyway). If any step fails, set `status = failed` +
    `last_error`, and let the user resume onboarding from wherever
    `status` indicates rather than starting over from scratch.
13. Add a "Settings" screen (can be minimal for this phase) that lets a
    user later change the repo connection, cloud selection, or region
    selection — reuses steps 6–11's components rather than being a
    separate implementation.

### Part C — Deployment sequencing
14. Write the `-infra` bundle: catalog/schema, secret scope declaration,
    Lakebase instance, App service principal.
    - **14a.** Include the secret-scope ACL grant (App service principal
      → `CAN WRITE`/`MANAGE`) — either as a declared bundle resource if
      the DAB schema supports it, or as a documented
      `databricks secrets put-acl` command in the deploy runbook (step
      16) if it doesn't. Verify against the current `databricks bundle
      schema` before assuming either way.
15. Write the `-app` bundle: the App resource itself (referencing the
    `-infra`-created service principal and secret scope by name/ID) plus
    its `app.yaml` declaring runtime env vars (secret-scope name,
    Lakebase connection details, GitHub App client ID). Deploy this once
    with **no real GitHub manifest yet** (Part A's manifest needs the
    host this deploy assigns — see the execution-order note above) so the
    App is live and its public host is known.
16. Write the deploy-order script/runbook: `bundle deploy` on `-infra` →
    confirm it's live (including the ACL grant from 14a if it's a
    separate command) → `bundle deploy` on `-app` → **explicitly start
    the App** (a bundle deploy alone can leave it stopped — this must be
    a distinct, scripted step, not assumed) → note the assigned public
    host → return to Part A step 1 to finalize and submit the real
    manifest with that host → redeploy `-app` if the manifest submission
    changed any App-side config.
17. Document (in this repo, e.g. a `docs/DEPLOY.md` or inline in each
    bundle's README) that `-ai-tools` does not exist yet and deploying it
    before Build Plan 01/03 land will fail by design.

## 6. Explicit fan-out map

**Preconditions (block everything else in this plan):**
- Part A cannot start until the repo owner has confirmed the GitHub
  org/account the App will be registered under (§2).
- Someone must decide and record the App framework choice (AppKit/React
  vs. a Python framework, per `databricks-apps`/`databricks-apps-python`)
  before Part B's UI components can be scoped as separate parallel
  tasks — the framework determines what "a separate file/component" even
  means. Treat "pick and record the framework" as step 0 of this plan.

**Can run in parallel once the framework choice is made:**
- Part A's *UI/backend code* (manifest-generation screen, callback
  endpoint skeleton, secret-writing logic — steps 1–5, minus actually
  submitting the real manifest to GitHub) and Part C's *bundle YAML*
  (steps 14–15) can be drafted in separate worktrees/sub-agent tasks —
  they touch different files (App backend code vs. `databricks.yml`/
  `app.yaml`). **However**, Part A cannot be *completed and verified*
  (the real manifest submitted, credentials actually written) until Part
  C step 14a's ACL grant and step 16's first `-app` deploy have landed —
  don't mark Part A "done" independent of that dependency; only its early
  scaffolding is truly parallel.
- Within Part B, once the shared `installations`/`tracked_clouds`/
  `tracked_regions` schema (§4) and the `status` state machine are agreed,
  the cloud-selection (step 10) and region-selection (step 11) UI
  components can be built by separate implementers as isolated
  components sharing that one state contract. They are **not** fully
  independent of the starting-point step (step 9) or of each other in
  the sense of "zero coordination" — all three render inside the same
  parent onboarding flow/router, so one implementer should own that
  parent shell (the step sequencing UI, the `status`-driven resume logic)
  while the others build against it as plug-in steps, rather than three
  people independently building three unrelated top-level flows.

**Must be sequential:**
- Part B's steps 6→7→7a→8 (installation picker → callback → repo
  resolution → token minting) are a strict pipeline — each depends on the
  prior step's output.
- Part A's "submit the real manifest" (finishing step 1–3) depends on
  Part C step 16 (first `-app` deploy → known public host) — this is the
  chicken-and-egg cycle described in the execution-order note at the top
  of §5; it is a *cross-part* dependency, not just an internal Part A
  ordering.
- Step 12 (mark onboarding `completed`) depends on steps 7–11 all
  succeeding, even though 10 and 11 can be built in parallel per above.
- Part C's step 16 (deploy-order script) depends on both step 14
  (including 14a) and step 15 existing.

**Cross-plan fan-out:** Build Plans 01 (RSS ingestion) and 02
(grounding corpus) **can** start adding their own tables/volumes to the
same shared `-infra` bundle in parallel with this phase's later steps,
once §4's schema and §3's bundle skeleton exist — they add to the
existing `-infra` bundle rather than replacing it. This is real
parallelization, not something to defer, but it does need explicit
coordination: don't let two sub-agents edit the same `databricks.yml`
concurrently — either serialize edits to that one file, or have one plan
own the file and the others submit patches to review before merging.

## 7. Platform constraints to build against (cross-review-confirmed)

- GitHub's callback URLs must be registered against the deployed App's
  **public `*.databricksapps.com` host** (or `*.azure.databricksapps.com`
  on Azure) — obtainable only *after* first deploy, not the workspace
  URL. This is exactly why §5's execution-order note and §6's fan-out map
  sequence Part C's first bare deploy before Part A's manifest can be
  finalized.
- The manifest callback (§5 step 3–4) needs the App's service principal
  to have **`CAN WRITE`/`MANAGE`** on the secret scope — the platform
  default is READ-only. §5 step 14a and the bundle-placement table (§3)
  call this out as an explicit grant, not something to assume comes for
  free.
- The App's filesystem is ephemeral — nothing from Part A or B may be
  cached to local disk between restarts. Every read of these credentials
  goes through the secret scope / Lakebase at request time.
- **Lakebase deploy-first schema ownership**: the App's service principal
  must deploy at least once before any local dev touches these tables —
  it creates and owns its Postgres schema on first deploy. Local dev
  against the schema before that first deploy fails with a bare `42501`
  permissions error. Document this explicitly in onboarding instructions
  for anyone building against this plan, so it isn't mistaken for a bug.

## 8. Validation checklist

- [ ] `-infra` bundle deploys cleanly from empty (fresh catalog/schema,
      fresh Lakebase instance, fresh secret scope with the ACL grant from
      step 14a applied, fresh service principal).
- [ ] `-app` bundle deploys cleanly after `-infra`, and the App starts
      (not just "deployed") without a manual extra step beyond the
      documented deploy script (§5 step 16).
- [ ] Admin can complete the GitHub App manifest flow once and see a
      "registered" status. **Re-running the flow while already
      registered shows the existing app's ID/name and does not create a
      second GitHub App or overwrite the existing secret values** — this
      is the defined behavior, not left to implementer discretion.
- [ ] A fresh installation can: connect a repo (confirm step 7a's repo
      resolution actually returns the right owner/name/default branch),
      choose "start from scratch" (confirm the repo is left completely
      unmodified — no commit created), pick 1–3 clouds, pick 0+ regions
      (confirm the UI doesn't force a region pick if the user has none
      configured yet, and only offers regions for already-selected
      clouds), and see the resulting `installations` row reach
      `status = completed` with matching `tracked_clouds`/
      `tracked_regions` rows.
- [ ] A second fresh installation, pointed at a *different* repo,
      produces an independent row with its own `github_installation_id`
      (UNIQUE constraint holds) — proves the per-installation model isn't
      accidentally singleton.
- [ ] Choosing "seed from starter pack" against the one seeded
      `starter_packs` row actually produces a real first commit (or PR,
      per whichever was chosen in step 9) in the newly-connected (empty)
      repo, and `installations.starter_pack_id` correctly FKs to that
      pack row.
- [ ] No PAT, private key, or client secret ever appears in application
      logs, the Lakebase tables, or the bundle YAML.
- [ ] Killing/restarting the App mid-flow leaves `installations.status`
      at whatever step last completed successfully (no partial writes to
      a later step's columns) — the user can resume onboarding from that
      `status` rather than the flow silently corrupting into an
      inconsistent-looking "completed" row.

## 9. Acceptance criteria (Definition of Done)

1. Both bundles that exist in this phase (`-infra`, `-app`) deploy in
   documented order via a single runbook command or script; `-ai-tools`
   is confirmed absent and its absence is documented, not accidentally
   half-built.
2. A brand-new GitHub org/repo can be connected end-to-end with zero
   manual GitHub-settings-page steps beyond the two unavoidable
   `github.com` consent clicks (the manifest-confirm screen and the
   install-picker — both by GitHub's design, not something this project
   can eliminate).
3. Cloud + region selections are queryable from Lakebase and are the
   values Build Plan 01 (RSS ingestion) will read to decide which
   feeds to poll.
4. The starter-pack registry has exactly one row ("Matt's version")
   pointing at this repo, and adding a second pack later requires only a
   row insert (confirm this by manually inserting a dummy second row in
   testing and verifying it appears as a second choice in the UI without
   a code change).
5. This build plan's own validation checklist (§8) passes.

## 10. Open items carried forward (not blockers, tracked for later plans)

- **App framework choice** (AppKit/React vs. a Python framework) — must
  be decided before Part B's components can be meaningfully parallelized
  (§6 precondition). Record the choice and resulting file layout here
  once made, since Build Plans 03/04/05 build on top of it.
- **Schema/migration mechanism** (§4) — raw SQL migration files, Alembic,
  or the chosen framework's own tool. Record the choice here once made;
  Build Plans 01/04/06 must reuse it rather than each picking their
  own.
- **Starting-point commit mechanism** (§5 step 9) — direct push vs. an
  opened PR for the starter-pack seed commit. Record whichever is chosen,
  and confirm Build Plan 04 (Review App) is consistent about whether
  *all* repo writes go through PRs or this first-commit case is a
  deliberate, documented exception.
- **Webhook events** (§5 step 1a) — this plan assumes no webhook events
  are consumed in Phase 0 (`hook_attributes.active: false`). If a later
  plan needs GitHub webhook events (e.g. reacting to a merged PR), that
  requires revisiting the manifest's `default_events` and building a
  webhook-signature-verified receiver endpoint — not in scope here.

---

**Changelog from cross-review:** fixed internal section cross-references
that pointed at the wrong section; corrected the Objective's overstated
"three-bundle" claim to match the rest of the plan (`-infra` → `-app`
only, `-ai-tools` absent); replaced the informal "one big transaction"
onboarding model with a `status`-driven state machine that doesn't assume
atomicity across external GitHub calls; filled in the exact GitHub App
Manifest flow mechanics (form POST, `code` exchange endpoint, required
manifest fields); added repo-resolution as an explicit step (GitHub's
callback doesn't hand back owner/name directly); added missing schema
fields/constraints (`status`, `default_branch`, `last_error`, uniqueness
constraints, per-cloud region scoping, stable `pack_id`); narrowed the
fan-out map's overstated independence claims (Part A/Part C only partly
parallel; cloud/region/starting-point share a parent flow); resolved a
contradiction where cross-plan fan-out was labeled "none" and then
described anyway; and closed out previously-untracked open items
(schema/migration mechanism, webhook scope).
