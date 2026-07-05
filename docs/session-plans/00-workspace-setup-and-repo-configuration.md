# Session Plan 00 — Workspace Setup / Repo Configuration (Phase 0)

**Source of truth:** `docs/PROJECT-PLAN.md` §1a, §1c, §1d, §3, §4, §5, Phase 0
section. This plan restates what's needed to build Phase 0; it does not
re-litigate design decisions already locked there — if something here
seems to conflict with `PROJECT-PLAN.md`, that file wins and this plan is
stale.

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
5. Deploy the `-infra` → `-ai-tools` → `-app` bundle sequence for the
   first time, proving the three-bundle dependency order end-to-end.

Everything in this phase is plumbing — no RSS ingestion, no research
agent, no review UI content yet. Done when a fresh installation can
authenticate to GitHub, pick a repo + starting point + cloud/region, and
land that config queryable in Lakebase, with the App itself reachable at
its public host.

## 2. Prerequisites

- None from other session plans — this is the first phase built.
- External: a GitHub organization/account where the one-time GitHub App
  registration will live (this is a decision for the repo owner, not an
  engineering task — confirm before starting Part A below).
- Databricks workspace with Unity Catalog enabled, permission to create
  catalogs/schemas, Lakebase enabled for the workspace/region, and
  Databricks Apps enabled.

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

| Resource | Bundle | Notes |
|---|---|---|
| UC catalog/schema (shared foundation for all phases) | `-infra` | Created once; every later phase's tables/volumes live under it |
| Secret scope for GitHub App credentials | `-infra` | Bundle declares the **scope only**; credential *values* are written at runtime by the App backend (§1a), never in bundle YAML |
| Lakebase database instance (+ branch, if used for dev/staging) | `-infra` | Backs all of §1c's OLTP tables, not just Phase 0's |
| Service principal — the App's own | `-infra` | Created here so grants in later bundles can target it |
| Databricks App resource | `-app` | Deployed **last** of the three bundles — see §7 below for the explicit post-deploy start step |

Phase 0 does not touch `-ai-tools` at all — nothing in this phase depends
on Vector Search or Genie. Confirm the `-infra` bundle's schema/volume
declarations are scoped generically enough that Phase 1/Phase 2/Phase 2b
can add their own tables/volumes to the *same* `-infra` bundle later
without restructuring it.

## 4. Data model (Lakebase)

Create these tables in the App's own Postgres schema (owned by the App's
service principal on first deploy — see §8 pitfall below). Exact DDL/ORM
choice is an implementation decision; the required fields are:

### `installations`
One row per Databricks workspace installation (there is exactly one row
in a self-hosted deployment, but design for multi-row from day one since
distributability is a hard requirement — see `PROJECT-PLAN.md` intro).

| Column | Type | Notes |
|---|---|---|
| `installation_id` | text/uuid, PK | This project's own row identifier — **not** the GitHub App installation ID (that's a separate column below) |
| `github_installation_id` | text | The ID GitHub returns from the App-installation callback |
| `github_repo_owner` | text | |
| `github_repo_name` | text | |
| `starting_point` | enum(`scratch`, `starter_pack`) | |
| `starter_pack_name` | text, nullable | Set only when `starting_point = starter_pack` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `tracked_clouds`
| Column | Type | Notes |
|---|---|---|
| `installation_id` | FK | |
| `cloud` | enum(`aws`, `gcp`, `azure`) | One row per selected cloud; multi-select via multiple rows |

### `tracked_regions`
| Column | Type | Notes |
|---|---|---|
| `installation_id` | FK | |
| `region` | text | e.g. `us-west-2`; one row per selected region |

### `starter_packs` (registry — seeded with one row at build time)
| Column | Type | Notes |
|---|---|---|
| `pack_name` | text, PK | `"Matt's version"` is the first row |
| `source_repo_owner` | text | |
| `source_repo_name` | text | |
| `source_ref` | text | e.g. `main` or a tag |

This table is the generic registry `PROJECT-PLAN.md` §1a calls for — adding
a second starter pack later is a row insert, not new code.

### `github_app_credentials` — **do not store here**
The app-level GitHub App credentials (private key, client secret, webhook
secret) are **secrets**, not Lakebase rows — they go in the Databricks
secret scope declared by `-infra` (§3 above), written at runtime per §5
Part A below. Do not create a Lakebase table for these.

> Tables for the RSS polling cursor, review-queue state, and the
> version-tracking projection (also called out in `PROJECT-PLAN.md` §1c)
> belong to Session Plans 01, 04, and 06 respectively — don't create them
> in this phase, but do confirm the schema/migration mechanism this phase
> establishes (§6 below) is reusable by those later plans without
> rework.

## 5. Task breakdown

### Part A — One-time GitHub App registration (app-level, not per-installation)
1. Build an admin-only setup screen in the App that generates a GitHub App
   **Manifest** (name, description, `contents:write` +
   `pull_requests:write` permissions scoped to repos the App will be
   installed on, and the App's own callback URL — see §8 platform
   constraint on which host to use).
2. Redirect the admin to GitHub's manifest-confirmation flow
   (`https://github.com/settings/apps/new?state=...` with the manifest
   POST'd per GitHub's documented flow).
3. Implement the callback endpoint that receives GitHub's temporary
   `code` and exchanges it (server-side, one-time) for: app ID, private
   key (PEM), webhook secret, client ID, client secret.
4. Write all five values to the Databricks secret scope from step 3 above
   — never to Lakebase, never to local/ephemeral disk (App filesystem is
   ephemeral — see §8).
5. Add a "GitHub App registered" status indicator to the admin screen so
   this doesn't need to be re-run per installation.

### Part B — Per-installation onboarding flow
6. Build a "Connect GitHub repo" button that redirects to GitHub's **App
   installation picker**, scoped to org/repo selection.
7. Implement the installation callback endpoint: receive the
   `installation_id`, write a new `installations` row (§4) with it plus
   the chosen `github_repo_owner`/`github_repo_name`.
8. Implement server-side installation-access-token minting (short-lived,
   on demand) using the app-level credentials from Part A + this
   installation's `github_installation_id` — no PAT anywhere in this
   flow.
9. Build the "starting point" step: scratch vs. starter pack, listing
   `starter_packs` rows. If starter pack chosen, copy that pack's
   `source_repo`@`source_ref` content into the newly-connected repo as
   its first commit (implementation detail: either a server-side git
   clone+push using the freshly-minted installation token, or an initial
   PR — pick whichever is simpler given the chosen App framework, but the
   result must be a real commit in the user's repo, not just a UI-only
   copy).
10. Build the cloud-selection step: multi-select AWS/GCP/Azure, default
    pre-checked = this installation's auto-detected deployment cloud
    (`WorkspaceClient().config.cloud` — confirmed working in a Databricks
    App's default runtime credentials per `PROJECT-PLAN.md` §6 #10).
    Write selections to `tracked_clouds`.
11. Build the region-selection step: free multi-select, **no pre-filled
    default** (per `PROJECT-PLAN.md` §6 #12 — no reliable auto-detection
    exists; best-effort account-API check only if those credentials
    happen to be configured, otherwise blank). Write selections to
    `tracked_regions`.
12. Persist all of the above as a single onboarding transaction (all-or-
    nothing — a half-completed onboarding should not leave a usable-looking
    but broken `installations` row).
13. Add a "Settings" screen (can be minimal for this phase) that lets a
    user later change the repo connection, cloud selection, or region
    selection — reuses steps 6–11's components rather than being a
    separate implementation.

### Part C — Deployment sequencing
14. Write (or confirm existing) `-infra` bundle: catalog/schema, secret
    scope, Lakebase instance, App service principal.
15. Write `-app` bundle: the App resource itself, referencing the
    `-infra`-created service principal and secret scope by name/ID.
16. Write the deploy-order script/runbook: `bundle deploy` on `-infra` →
    confirm it's live → `bundle deploy` on `-app` → **explicitly start the
    App** (a bundle deploy alone can leave it stopped — this must be a
    distinct, scripted step, not assumed).
17. Document (in this repo, e.g. a `docs/DEPLOY.md` or inline in each
    bundle's README) that `-ai-tools` does not exist yet and deploying it
    before Session Plan 01/03 land will fail by design.

## 6. Explicit fan-out map

**Can run in parallel (independent implement tasks, no shared file
conflicts if scoped carefully):**
- Part A (GitHub App registration: steps 1–5) and Part C's bundle-writing
  (steps 14–15) are independent of each other — one is App backend code,
  the other is bundle YAML. Can be dispatched as two separate `implement`
  tasks in parallel worktrees.
- Within Part B, the cloud-selection (step 10) and region-selection (step
  11) UI components are independent of each other and of the starting-
  point step (step 9) — three parallelizable sub-tasks once the
  `installations`/`tracked_clouds`/`tracked_regions` schema (§4) is
  agreed, since none of the three reads what the others write.

**Must be sequential:**
- Part B's steps 6→7→8 (installation picker → callback → token minting)
  are a strict pipeline — each depends on the prior step's output.
- Part A must fully complete (secrets exist) before Part B's step 8
  (token minting) can be implemented against real credentials, though the
  *UI* for steps 6–7 can be built and tested with mocked credentials in
  parallel with Part A.
- Part C's step 16 (deploy-order script) depends on both step 14 and step
  15 existing.
- Step 12 (persist as one transaction) depends on steps 9, 10, and 11 all
  being implemented, even though those three can be built in parallel.

**Cross-plan fan-out:** none — this is the first session plan, nothing
upstream to parallelize against. Session Plans 01 (RSS ingestion) and 02
(grounding corpus) can start their own `-infra` bundle additions in
parallel with this phase's later steps once §4's schema and §3's bundle
skeleton exist, since they add tables/volumes to the same `-infra` bundle
rather than replacing it — coordinate to avoid two sub-agents editing the
same `databricks.yml` simultaneously (serialize that one file's edits, or
have one plan own the file and the other submit a patch to review).

## 7. Platform constraints to build against (cross-review-confirmed)

- GitHub's callback URLs must be registered against the deployed App's
  **public `*.databricksapps.com` host** (or `*.azure.databricksapps.com`
  on Azure) — obtainable only *after* first deploy, not the workspace
  URL. This means Part A's manifest generation (step 1) can't be fully
  finished until the App has been deployed once with a placeholder/no-op
  callback, then updated with the real host. Sequence: deploy `-app`
  once → note the assigned host → complete Part A's manifest with that
  host → redeploy.
- The manifest callback (step 3) needs the App's service principal to
  have **CAN WRITE/MANAGE** on the secret scope — the platform default is
  READ-only. This must be an explicit grant in the `-infra` bundle or a
  documented one-time `databricks secrets put-acl` step, not assumed.
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
      fresh Lakebase instance, fresh secret scope, fresh service
      principal).
- [ ] `-app` bundle deploys cleanly after `-infra`, and the App starts
      (not just "deployed") without a manual extra step beyond the
      documented deploy script.
- [ ] Admin can complete the GitHub App manifest flow once and see a
      "registered" status; repeating it is either idempotent or
      explicitly blocked with a clear message.
- [ ] A fresh installation can: connect a repo, choose "start from
      scratch," pick 1–3 clouds, pick 0+ regions (confirm the UI doesn't
      force a region pick if the user has none configured yet), and see
      the resulting `installations`/`tracked_clouds`/`tracked_regions`
      rows land correctly.
- [ ] A second fresh installation, pointed at a *different* repo,
      produces an independent, non-colliding set of rows — proves the
      per-installation model isn't accidentally singleton.
- [ ] Choosing "seed from starter pack" against the one seeded
      `starter_packs` row actually produces a real first commit in the
      newly-connected (empty) repo.
- [ ] No PAT, private key, or client secret ever appears in application
      logs, the Lakebase tables, or the bundle YAML.
- [ ] Killing/restarting the App mid-flow does not lose in-flight
      onboarding state in a way that corrupts (rather than just requiring
      a retry of) the `installations` row.

## 9. Acceptance criteria (Definition of Done)

1. All three bundles (`-infra` and `-app`; `-ai-tools` does not exist yet
   and that's expected) deploy in documented order via a single runbook
   command or script.
2. A brand-new GitHub org/repo can be connected end-to-end with zero
   manual GitHub-settings-page steps beyond the two unavoidable
   `github.com` consent clicks (§1a's "honest caveat").
3. Cloud + region selections are queryable from Lakebase and are the
   values Session Plan 01 (RSS ingestion) will read to decide which
   feeds to poll.
4. The starter-pack registry has exactly one row ("Matt's version")
   pointing at this repo, and adding a second pack later requires only a
   row insert (confirm this by manually inserting a dummy second row in
   testing and verifying it appears as a second choice in the UI without
   a code change).
5. This session plan's own validation checklist (§8) passes.

## 10. Open items carried forward (not blockers, tracked for later plans)

- The exact framework (AppKit/React vs. a Python framework) for the App
  is not decided in `PROJECT-PLAN.md` — Session Plan 00's implementer
  should pick per `databricks-apps`/`databricks-apps-python` guidance and
  note the choice here once made, since Session Plans 03/04/05 build on
  top of whatever's chosen.
- Whether the "seed from starter pack" initial commit (step 9) is a
  direct push or an opened PR is left as an implementation choice — note
  whichever is chosen, since Session Plan 04 (Review App) should be
  consistent about whether *all* repo writes go through PRs or whether
  this one first-commit case is a deliberate exception.
