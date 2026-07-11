# aiSkillUpdater — Project Plan

**Goal:** turn the current manual "run `/investigate` in Polly, review the
diff, copy files by hand" workflow (see `docs/STAGING-AREA.md`) into an
**event-driven Databricks App** that watches the Databricks Release Notes RSS
feed, researches each announcement, drafts skill updates, and lets a human
review + apply them to their local coding-agent skill directories — with a
human approval gate before anything ships.

**Distributable, not single-tenant:** this app is meant to be installable by
other people into their own Databricks workspaces, not just run against
*this* repo. Each installation configures its own GitHub repo as the place
it reads from and opens PRs against — see §1a.

## 0. Current state (baseline)

- This repo is the staging area: one directory per skill, each with an
  updated `SKILL.md` / `references/` and a `REVIEW-BEFORE-COPYING.md` diff
  summary.
- Audits today are triggered manually and run by ad hoc sub-agents that
  cross-check each `SKILL.md` against live `docs.databricks.com` pages.
- The skills a user codes against locally come from `databricks aitools
  install` (CLI subcommand, shipped since CLI v1.5.0). That command:
  - supports a **global** (all-users) or per-user install,
  - **auto-detects** which local AI models/tools the caller has (Claude
    Code, Cursor, Codex CLI, OpenCode, GitHub Copilot, Antigravity) and
    installs the matching skill variant for each,
  - is a point-in-time snapshot: it does not itself track drift against
    current docs, so CLI-shipped skills go stale between CLI releases too.

Everything below replaces "manual, on-demand, batch-diff-against-docs" with
"event-driven, RSS-triggered, per-announcement research, human-reviewed,
locally-applied" — the human review step stays, it just gets a proper UI.

## 1. Target architecture

```
┌───────────────────────────────────────────────────────────────────┐
│ RSS Ingestion (event-driven, not pure polling)                     │
│  - Per-installation cloud selection (§1a): the user picks which of    │
│    AWS/GCP/Azure feeds (§6 #6) to track, any combination, defaulting  │
│    to the cloud this installation is deployed on                      │
│  - Plain RSS 2.0 has no push/webhook primitive at all (no advertised  │
│    WebSub/PubSubHubbub hub on any of the 3 feeds, confirmed live) —   │
│    delivery is always "a Job fetches feed.xml on a schedule," never   │
│    server-initiated (§6 #6); a scheduled **Databricks Job** (not the  │
│    App — see compute-owner note below) does the fetch, using          │
│    conditional GET (`If-None-Match` on the feed's `ETag`) so a        │
│    no-change poll costs a ~1KB `304` not a ~1MB re-download; **daily**│
│    cadence is enough since every `<pubDate>` across all 3 feeds is    │
│    stamped `00:00:00 GMT` (day-granular, ~72% of recent days post)    │
│  - No Zerobus: this is pull, not push — the Job parses the fetched    │
│    XML in-process and does a plain batch append (Spark/pandas write,  │
│    no gRPC stream) straight into `rss_bronze` — raw, append-only, no  │
│    dedup logic at this layer, exactly as received                     │
│  - A silver step (`rss_silver`) de-dupes on `<guid>` (identical to    │
│    `<link>` in these feeds — the correct dedup key, not title+link)   │
│    within a time window keyed off pubDate (bounded state, not a       │
│    global keep-first) so a cross-cloud announcement (2-3 clouds       │
│    selected) doesn't trigger the pipeline more than once              │
│  - Arrival of a new `rss_silver` row is the trigger, via a Databricks │
│    Jobs **Table update** trigger on `rss_silver` (`trigger.table_update`, │
│    `condition: ANY_UPDATED`) — resolved §6 #13; not `file_arrival`,     │
│    which only covers Volumes/external locations, not managed tables      │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ Research Agent (Job task, one run per new rss_silver row)           │
│  1. LIVE fetch of the announcement body + every link it references — │
│     never wait on the next scheduled corpus refresh, since same-day  │
│     announcements often link to pages not yet indexed                │
│  2. Pull current docs.databricks.com context for the subject         │
│     (Vector Search index) as supplementary grounding                  │
│  3. ai_query (not ai_summarize — needs cross-document reasoning over    │
│     multiple fetched URLs, not single-doc condensation) to synthesize   │
│     what changed with inline footnote-style citations, returning a      │
│     structured summary + claims + citations, not free-text prose        │
│  4. ai_classify (multilabel) the announcement against the taxonomy of  │
│     installed skills (skill name + description = the label set, PLUS   │
│     an always-present "New Feature" bucket) to get the affected         │
│     skill(s)/bucket labels — confidence/rationale are NOT native        │
│     ai_classify output, so compute those via a separate lightweight     │
│     LLM step if the review UI needs to show them                        │
│  5. ai_extract cloud availability (AWS/Azure/GCP) and, for CSP          │
│     workspaces, compliance-level availability (No CSP, HIPAA,           │
│     PCI-DSS, FedRAMP Moderate) for the feature, from the fetched         │
│     source(s) — see §1b for the target skill-content format             │
│  6. For each affected skill (or "New Feature" candidate): draft a       │
│     proposed SKILL.md/references patch (unified diff) grounded in       │
│     the summary + citations + step 5's availability findings            │
│  7. Write findings to a JSON file (one per rss entry) in a UC Volume —  │
│     summary, citations[], skill_classifications[], availability{},      │
│     proposed_diffs[], status ("pending_review") — designed to be         │
│     auto-loaded by a Lakeflow Declarative Pipeline (streaming,           │
│     Autoloader-based) into                                              │
│     `research_log` and the Vector Search index, rather than written      │
│     directly to Delta from the agent itself                              │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ Ingestion Pipeline (Lakeflow Declarative Pipeline, streaming)       │
│  - Autoloader watches the JSON findings volume → bronze → silver     │
│    (`research_log` Delta table, normal write, no custom Sink); a      │
│    Delta Sync Vector Search index (managed embeddings, TRIGGERED)     │
│    exists on `research_log`, and the pipeline calls `sync_index()`    │
│    after each batch (TRIGGERED needs an explicit call, not automatic) │
│  - Decouples "agent writes a file" from "data lands in queryable      │
│    tables + search index", so the Research Agent task stays simple    │
│    and idempotent (just write a JSON file)                            │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ Databricks App (AppKit + React, thin backend)                       │
│  Review queue:                                                       │
│   - One card per research_log row: summary + footnoted citations,     │
│     classified skill(s) + confidence, proposed diff per skill          │
│   - Per-skill accept / reject / edit-then-accept                       │
│   - Full markdown viewer + editor for any skill file (view current,   │
│     view proposed, edit either, save)                                  │
│  On accept:                                                            │
│   - Writes the approved version into the configured skills repo (§1a)   │
│     (commit + PR, never direct-to-main — same as before) AND/OR         │
│   - Offers "Apply locally now" → runs the Local Installer (§4) against  │
│     the machine the App can reach (see open question in §6)             │
│  Skill Library browser:                                                │
│   - Search/browse all skills + their version history                    │
│   - Compare: user's maintained version vs. CLI-bundled version vs.      │
│     latest research-proposed version, three-way                         │
│   - "Check for latest from <starter pack>" (per-skill or full sweep) →  │
│     diffs against the pack's source repo, feeds result(s) into the       │
│     SAME review queue above for accept/reject/edit-then-accept (§1a)     │
│  CLI-drift check (on new CLI release, see §2 Phase 4):                  │
│   - Table: skill, user's base_cli_version, user's current version,      │
│     CLI's shipped version in the new release, verdict                   │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (only on explicit user action)
┌───────────────────────────────────────────────────────────────────┐
│ Local Installer — git-based (primary) + downloadable script (fallback) │
│  - Primary: the configured skills repo's PR (§1a, already produced by   │
│    Phase 3's accept flow) is the trust boundary. User runs `git pull` +  │
│    a small, reviewed-once `apply-skill.sh` (checked into the same repo,  │
│    seeded there by the starter pack or added on first setup) that fans   │
│    out the approved skill into each detected agent's directory (Claude   │
│    Code, Cursor, Codex, OpenCode, GitHub Copilot, Antigravity — the       │
│    CLI's own `--agents` set — NOT via `aitools install` itself, see      │
│    note below; Pi is out of scope for now)                                │
│  - Fallback: App also offers a downloadable one-off script (signed,      │
│    short-TTL URL, writes files only, shown to the user before it's        │
│    piped to a shell) for users without the configured repo cloned         │
│    locally                                                                │
│  - Every file written gets the version-tag frontmatter (see §5) so        │
│    drift is always re-detectable                                          │
│  - NOTE: `databricks aitools install` cannot be reused/piggybacked —      │
│    it has no source-override flag (hardcoded to                          │
│    github.com/databricks/databricks-agent-skills). This project needs    │
│    its own installer regardless of agent set; not this phase's           │
│    mechanism.                                                             │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (optional, later phase)
┌───────────────────────────────────────────────────────────────────┐
│ Skill Knowledge Base + MCP Server                                    │
│  - Same Vector Search index + research_log, exposed as:                │
│    - full-text/semantic search UI in the App                           │
│    - a Databricks App-hosted MCP server so OTHER agents (not just       │
│      this app's UI) can query "what's the latest on X" or "which        │
│      skill covers Y" programmatically                                   │
└───────────────────────────────────────────────────────────────────┘
```

A human always approves before a skill file changes anywhere — in the repo
PR or on their own machine. Nothing auto-applies.

## 1a. Skill repo configuration (bring-your-own-repo)

Every box above that says "this repo" is really shorthand for **the
configured skills repo for this installation** — a per-workspace setting,
not a hardcoded value. That's what makes the app installable by someone
else into their own workspace without forking this repo or getting push
access to `mkgs-databricks-demos/aiSkillUpdater`.

- **Setup step, once per installation**: during first-run/onboarding, the
  App asks the user (or workspace admin) to connect a GitHub repo — owner +
  repo name, plus credentials via a **GitHub App connection** (resolved —
  see the provisioning flow below and §6 open question #8). That repo
  becomes the single target this installation reads skills from and opens
  PRs against for the lifetime of the install (changeable later in
  settings, but not silently).
- **Starting point options at setup**, since a brand-new repo is empty:
  1. **Start from scratch** — the user's repo stays empty until the first
     accepted proposal creates its first skill directory.
  2. **Seed from a starter pack** — copy an existing, curated skill set
     into the user's new repo as the initial commit. "**Matt's version**"
     (this repo, `mkgs-databricks-demos/aiSkillUpdater`) is the first
     available starter pack; the mechanism should be generic enough to add
     more named packs later without new code, just a registry entry
     (pack name → source repo + ref).
  - Either way, after setup the installation is fully independent — it
    reads/writes only its own configured repo from then on. A starter pack
    is a one-time seed, not an ongoing dependency.
- **GitHub connection mechanism: GitHub App, fully provisioned from inside
  the Databricks App's own UI** (resolves §6 open question #8). Two
  distinct provisioning steps, both driven by a button-click-and-redirect
  in the App — no manual GitHub settings-page form-filling, no hand-copied
  client secrets or private keys, no CLI steps outside the App:
  1. **One-time: register the GitHub App itself** (per deployment of this
     project, not per end user). An admin setup screen in the App uses
     GitHub's **App Manifest flow**: the App generates a manifest (name,
     permissions — `contents:write`, `pull_requests:write` on the target
     repo(s) — and its own callback URL), the admin clicks through to
     GitHub to confirm/name the app, and GitHub redirects back with a
     temporary code. The App exchanges that code server-side for the full
     credential set (app ID, private key (PEM), webhook secret, client
     ID/secret) and stores them as Databricks secrets. The admin never
     manually fills in a GitHub App settings form or copy-pastes a private
     key.
  2. **Per-installation: connect a repo**. From the per-workspace setup
     screen (this section), the user clicks "Connect GitHub repo," which
     redirects to GitHub's own **App installation** picker (scoped to
     choosing an org/repo to install the app on) and back to the App with
     an `installation_id`. The App stores that ID against this
     installation's config in **Lakebase** (§1c — not a secret, the ID
     alone isn't a credential; not Delta either, this is per-installation
     OLTP state) and mints short-lived **installation access tokens**
     on-demand server-side whenever it needs to commit or open a PR. No
     PAT, and no long-lived credential the user has to generate or paste
     themselves.
  - **Honest caveat**: GitHub requires the human's consent click to happen
     on `github.com` itself for both flows (the manifest confirm screen,
     and the install-picker) — that's an unavoidable, by-design part of
     how GitHub Apps work, not something this project can eliminate. The
     part that stays entirely inside the Databricks App's UI is
     everything else: no manual form-filling, no copying credentials by
     hand, no separate CLI or GitHub-settings-page workflow — just
     click-redirect-confirm-return.
  - This adds real Phase 0 implementation surface: two callback endpoints
     in the App backend (manifest-creation callback, installation
     callback), secret storage for the app-level credentials, and a small
     per-installation state table for `installation_id` → workspace repo
     config. Sized accordingly when Phase 0 gets scoped for build.
- **Resolved — starter packs stay live reference sources, not just a
  one-time seed**: after setup, a user can hit a **"Check for latest from
  Matt's version"** action (Skill Library browser, per-skill or a full
  sweep) that diffs their current skill(s) against the pack source repo's
  current content and produces the same **accept/reject/edit-then-accept**
  review flow already used for RSS-driven proposals (Phase 3) — reusing
  that review queue rather than inventing a second one. This means Phase
  0's pack registry must persist a live `source repo + ref` per
  installation (not just perform a point-in-time copy at setup), so it can
  be re-diffed on demand.
- **Cloud tracking selection** (also per-installation config, alongside the
  repo connection above): the user chooses which of the three Release
  Notes feeds (§6 #6 — AWS/GCP/Azure) this installation ingests, any
  combination including all three, defaulting to **the cloud this
  installation is deployed on** (auto-detected via
  `WorkspaceClient().config.cloud` — resolved, see §6 #10). Changeable
  later in settings, same as the repo connection.
- **Region tracking selection**, alongside cloud selection: since
  compliance-standard availability (§1b) is itself region-scoped (the
  Databricks compliance docs publish Feature × Region matrices, not a
  single yes/no per standard), the user also picks **which region(s),
  within their selected cloud(s), they actually care about** for
  feature-availability tracking — not "all regions everywhere," which
  would bloat every skill's availability data with regions the user will
  never deploy to. **Default resolution (§6 #12): no reliable universal
  auto-detection exists** — unlike cloud (§6 #10), the Databricks SDK
  exposes no workspace-scoped region field; the only authoritative source
  is the account-level Workspaces API (AWS/GCP) or Azure Resource
  Manager, both of which need account/management-plane credentials a
  Databricks App doesn't have by default. So: **best-effort** — try the
  account API if the installation happens to have those credentials
  configured (unlikely for most installs); otherwise **prompt the user
  to pick their region(s) explicitly at setup**, with no pre-filled
  default, rather than guessing from the workspace hostname (explicitly
  unreliable for modern workspace URLs, confirmed by investigation).
  Multi-select supported (a user managing workloads in two AWS regions
  can track both). Changeable later in settings, same as cloud selection.
- These two selections (cloud + region) are the pieces of Phase 0 config
  that reach into the RSS/Research Agent pipeline (§1, Phase 2) and the
  availability model (§1b) — everything else there
  remains workspace infra untouched by which skills repo is configured.
- This has no effect on Phase 4's CLI-drift logic (still comparing against
  the public `databricks-agent-skills` repo, independent of which skills
  repo this installation is configured to use) — it only changes *which
  repo* Phase 3's accept flow and Phase 5's Local Installer target, and
  (via the cloud-tracking selection above) *which RSS feeds* Phase 2
  ingests for this installation.

## 1b. Skill content: cloud + compliance availability

Skills should state not just *what* a feature does but *where it's usable*
— by cloud, and for CSP (Compliance Security Profile) workspaces, by
compliance standard. Two dimensions, both extracted by the Research Agent
(§1, step 5) from the same fetched sources used for the summary/citations:

- **Cloud availability**: AWS / Azure / GCP, independent of which feeds
  this installation happens to track (§1a) — a feature announced only on
  the AWS feed but later confirmed available on Azure too should still
  record Azure availability if the fetched docs say so.
- **Compliance-level availability**, for CSP workspaces specifically:
  **No CSP** (standard workspace), **HIPAA**, **PCI-DSS**, and **FedRAMP
  Moderate**. Feature availability at these compliance levels can lag
  general availability significantly, and is exactly the kind of detail a
  human coding against a skill would otherwise have to go re-check by hand
  — which is the whole reason this project exists.
- **Region tracking** (§1a): since compliance-standard support is itself
  region-scoped, not just standard-scoped (the Databricks compliance docs
  publish Feature × Region matrices per standard, not a single yes/no), the
  user selects which region(s) within their tracked cloud(s) they care
  about. Availability is tracked **per tracked region**, not flattened
  into one boolean per standard — see the format below.
- **Format** (revised to be region-scoped): each skill carries a
  structured **availability block** (frontmatter, so it's both
  machine-filterable and renders in the App's Skill Library browser)
  alongside a human-readable summary in the skill body:
  ```yaml
  availability:
    clouds: [aws, azure, gcp]           # or a subset
    compliance:                          # keyed by this installation's tracked regions only (§1a) — not every region Databricks supports
      no_csp:
        us-west-2: true
        us-east-1: true
      hipaa:
        us-west-2: true
        us-east-1: true
      pci_dss:
        us-west-2: true
        us-east-1: unknown              # not yet confirmed either way
      fedramp_moderate:
        us-west-2: true
        us-east-1: false
  ```
  `unknown` (not just `true`/`false`) matters here: the Research Agent
  should never assert compliance availability it didn't actually find
  evidence for — absence of a mention is not evidence of unavailability.
  Only tracked regions are populated — if the user later adds a new
  tracked region, existing skills' availability blocks gain a new key
  (re-extracted, not assumed) for that region rather than the schema
  needing to change.
- **Extraction mechanism (resolved, §6 #11)**: Databricks publishes
  structured, scrapable **"Regional support for features" HTML tables**,
  one per compliance standard (Feature × Region grid) — not a JSON/API
  feed, but consistent server-rendered `<table>` markup on fixed URLs:
  - HIPAA: `docs.databricks.com/aws/en/security/privacy/hipaa`
  - PCI-DSS: `docs.databricks.com/aws/en/security/privacy/pci`
  - FedRAMP Moderate: `docs.databricks.com/aws/en/security/privacy/fedramp`
    (**AWS-only** — no FedRAMP page/authorization exists for GCP or Azure;
    `fedramp_moderate` should only ever be assessed for AWS-hosted
    workspaces, always `false`/n\a otherwise, never `unknown`)
  - Plus a cross-feature preview/beta support table on the main CSP page
    (`docs.databricks.com/aws/en/security/privacy/security-profile`) for
    features still in preview.
  These three standard pages are the **primary, most authoritative
  source**, and their Feature × Region shape is exactly what the
  region-scoped extraction above needs — `ai_extract` (or plain
  table-parsing, arguably more reliable than an LLM call for a fixed HTML
  table) reads a feature's row, filtered down to just this installation's
  **tracked regions** (§1a), across all three pages. This is a **periodic
  reference fetch**, not something tied to a specific RSS announcement's
  linked URLs — the Research Agent should check these three fixed pages
  for the classified feature regardless of what the announcement itself
  links to. Individual feature/release-notes pages *sometimes* also carry
  their own compliance notes (e.g. the Model Serving limits page has its
  own region × standard table), but coverage there is inconsistent/ad hoc
  — useful as a secondary confirmation when an announcement happens to
  link to one, not something to rely on as a primary source.
  - The FedRAMP Moderate page itself warns features may be listed as
    available before they're actually released — treat as directional,
    not a hard guarantee, and say so in any skill content that cites it.
  - If a tracked region isn't a compliance standard's authorization
    boundary at all (e.g. a tracked `eu-west-1` region against FedRAMP
    Moderate, which only covers 4 `us-*` regions), record that region's
    value as `false`, not `unknown` — the boundary itself is documented
    and definitive, unlike an actual unconfirmed feature-level gap.
  - **If `ai_extract` is used instead of plain table parsing** (cross-
    reviewed vs. `databricks-ai-functions`): the tri-state `true`/`false`/
    `unknown` values should be modeled as an **enum of strings**, not
    booleans (`ai_extract`'s schema doesn't have a native tri-state
    boolean); keep the schema within documented limits (≤128 fields,
    ≤7 nesting levels) by scoping it to just the tracked regions/clouds
    for this installation rather than every region Databricks supports;
    and access the returned VARIANT via `result:response:...` path
    syntax, not plain dot notation.
- **Not a one-time concern**: like Phase 4's CLI-drift check, compliance/
  cloud availability can change independent of a new RSS announcement
  (e.g. FedRAMP authorization catching up later for an already-GA
  feature). A periodic re-check of existing skills' availability blocks
  against current docs is a plausible future phase, not required for MVP
  — tracked as a stretch idea, not yet a numbered phase.

## 1c. State storage: Lakebase for OLTP config/state

**Cross-review finding (real design gap, not a refinement)**: the plan
previously stored *everything* — including small, per-installation,
frequently-mutated config — in Delta tables. Both `databricks-apps` and
`databricks-lakebase` route exactly that profile ("app-specific
metadata/config," "low-latency transactional CRUD," "user session data")
to **Lakebase** (autoscaling Postgres, scale-to-zero), not Delta, which is
designed for append/analytical workloads with high per-commit latency and
no efficient single-row update.

- **Moves to Lakebase**:
  - The GitHub App `installation_id` → repo-config mapping (§1a)
  - The starter-pack registry (pack name → source repo + ref, persisted
    for on-demand re-diff per §1a's "live reference source" design)
  - Cloud + region tracking selections (§1a)
  - The RSS polling cursor per tracked feed (`ETag`/`Last-Modified` from
    the last successful conditional-GET, per §2 Phase 2) — small,
    frequently-overwritten, single-row-per-feed state, the same profile
    as everything else on this list
  - Review-queue state: per-skill accept/reject/edit-then-accept status,
    classification overrides, and user-created taxonomy buckets (Phase 3)
  - The version-tracking table used for Phase 4's three-way CLI-drift
    comparison (§5) — the `SKILL.md` frontmatter in the repo remains the
    **authoritative source of record**; the Lakebase table is its
    queryable projection for the App to read/write interactively
- **Stays in Delta/UC — explicitly NOT moved**: `rss_bronze`, `rss_silver`,
  `research_log`, the docs corpus, and the Vector Search source tables.
  These are append-only/Autoloader-streamed/analytical, and `research_log`
  backs the Delta Sync index (§2 Phase 2b) — moving them to Lakebase would
  break the streaming ingestion and the managed Vector Search sync.
- **Optional bridge, not a replacement**: if the Phase 3 review-queue UI
  needs `research_log` rows at operational (sub-second) latency rather
  than warehouse-query latency, use a **Lakebase synced table** (read-only
  mirror of `research_log`) rather than relocating the source. Caveats to
  design around: synced tables are **read-only** in Postgres (writes
  corrupt the sync), the App's service principal needs an **explicit
  GRANT** to read them (`CAN_CONNECT_AND_CREATE` alone isn't enough),
  row/column-level access control is **not propagated** through the sync,
  and creation goes through `databricks postgres create-synced-table`
  (not a DAB `synced_database_tables` resource, per §1a's use of
  autoscaling Lakebase). Default to it only if plain warehouse Analytics
  reads (§2 Phase 3) prove too slow in practice — don't reach for it
  up front.
- **Deploy-first schema ownership**: per Lakebase's model, the App's
  service principal must **deploy before any local dev** against these
  tables, since it creates and owns its own Postgres schema on first
  deploy. Running local dev against the schema before that first deploy
  fails with a permissions error (`42501`), not a helpful "not set up yet"
  message — worth calling out explicitly in Phase 0's build tasks so it
  isn't a surprise.

## 1d. Bundle deployment design (three bundles, dependency-ordered)

**Repo owner preference, locked in as the deployment design**: this project
ships as **three separate DABs**, not one, deployed in a strict dependency
order. Each is its own `databricks.yml` (own directory, own bundle root),
not three targets of a single bundle — because the ordering constraint is
"fully deployed" gating, not just "deployed to a different workspace,"
which is what DAB targets are for.

- **`-infra`** — everything that must exist *before* the vector store or
  the App can be deployed. This is the foundation layer: nothing in it
  depends on data already being in Unity Catalog.
  - UC catalog/schema, the docs-corpus and JSON-findings UC Volumes
    (§ Phase 1, § Phase 2b)
  - Secret scope for GitHub App credentials (§ Phase 0) — the scope
    itself only; per §1a's provisioning flow, the actual credential
    *values* are written at runtime by the App, never declared in bundle
    YAML
  - Lakebase database instance/project (and branch, if used for
    dev/staging) backing §1c's OLTP state tables
  - Service principals: the App's own, and any dedicated principal for
    the RSS polling Job (§2 Phase 2) — created here so grants can target
    them from resources deployed later
  - Target table DDL: `rss_bronze` (§2's full `CREATE TABLE` — CDF, Row
    Tracking, the `feed_value VARIANT` column, Variant Shredding,
    `CLUSTER BY AUTO`), and any other raw landing tables
  - The Lakeflow Declarative Pipelines that get data ready for use:
    `rss_bronze` → `rss_silver` dedup (§2), and the findings-ingestion
    pipeline through `research_log` (§ Phase 2b) — both produce the
    Delta tables that `-ai-tools`'s Vector Search index syncs from, so
    they must exist and have run at least once before that index is
    created
  - The RSS polling Job itself (§2) — it only ever writes to
    `-infra`-owned tables, so it belongs here rather than in `-ai-tools`
- **`-ai-tools`** — resources that need some `-infra` piece already
  deployed (and, in most cases, populated) before they themselves can be
  created. Deploying `-ai-tools` before `-infra` has run at least once is
  expected to fail, not just be suboptimal.
  - The Vector Search endpoint and both indexes (docs corpus, § Phase 1;
    `research_log`, § Phase 2b) — both require their source Delta tables
    to already exist in UC. The endpoint and indexes ARE declarable bundle
    resources (`resources.vector_search_endpoints` /
    `resources.vector_search_indexes`, confirmed in the v1.5.0 bundle
    schema) — declare them; an idempotent post-deploy setup job is a
    contingency only, not the default
  - Genie Spaces, if/when added (§ Phase 6 stretch) — same
    data-must-exist-first constraint as Vector Search
  - Any external MCP tools registered against Unity Catalog objects with
    their own UC permissions/grants (§ Phase 6 stretch) — registration
    requires the underlying UC objects (tables, the Vector Search index)
    to already be present
  - The CLI-drift-detection Job (§ Phase 4) is a judgment call: it reads
    from Vector-Search-adjacent state but doesn't itself touch the index,
    so it can live in `-ai-tools` for now since it's dependent on the
    version-tracking Lakebase table (`-infra`) rather than on anything
    `-ai-tools` itself creates — revisit if that changes
- **`-app`** — the Databricks App resource itself (§ Phase 0, Phase 3,
  Phase 5), plus the explicit post-`bundle deploy` "start the App" step
  already noted below, since a bundle deploy alone can leave it stopped.
  Deployed last because Phase 3's Review App reads `research_log`
  (through the Vector Search index or a warehouse query) and Phase 5's
  Local Installer both assume the earlier two bundles are fully live.
- **Why three and not one**: a single bundle can't express "don't attempt
  to create this Vector Search index until that Delta table has rows in
  it" — DABs deploy declared resources, they don't sequence around
  runtime data readiness. Three bundles deployed in order (`-infra` →
  run the Job/pipelines at least once → `-ai-tools` → `-app`) makes that
  ordering an operational script step instead of something the bundle
  schema would have to (and can't) express. Each bundle is independently
  re-deployable afterward — a Phase 7 alert fix only needs a fresh
  `-infra` deploy, not a full three-bundle re-run.
- Every earlier note in this doc that referred to "the DAB" (singular)
  now refers to whichever of these three actually owns that resource —
  see the per-phase bundling notes below (Phase 0/1/2b/3) for the
  specific split.

## 2. Build phases

### Phase 0 — Workspace setup / repo configuration
- First-run onboarding flow per §1a: connect a GitHub repo via a **GitHub
  App**, choose "start from scratch" vs. "seed from a starter pack" (with
  "Matt's version" as the first available pack), pick tracked cloud(s)
  (§1a, defaulting to this installation's auto-detected deployment cloud)
  + tracked region(s) (§1a, §6 #12 — no reliable auto-default, user picks
  explicitly), and store the resolved repo + installation config.
- Two-part GitHub App provisioning (resolved, §1a and §6 #8), both build
  tasks for this phase:
  1. **App-level, one-time**: an admin setup screen implementing GitHub's
     App Manifest flow (generate manifest → redirect to GitHub to confirm
     → callback exchanges the temporary code for app ID, private key,
     webhook secret, client ID/secret → store as Databricks secrets). Only
     run once per deployment of this project, not per installation.
  2. **Per-installation**: a "Connect GitHub repo" button that redirects to
     GitHub's App installation picker and back with an `installation_id`,
     stored in **Lakebase** (§1c). Installation access tokens are minted
     on-demand server-side from then on — no PAT ever stored or handled.
- **Three Databricks platform constraints confirmed by cross-review** that
  the build tasks above need to account for:
  - GitHub's redirect/callback URLs must be registered against the
    deployed App's **public `*.databricksapps.com` host** (Azure:
    `*.azure.databricksapps.com`) — obtained only after first deploy, not
    the workspace URL. Custom `GET` callback routes themselves are
    supported (via AppKit's `server.extend()`/`onPluginsReady`, or the
    chosen Python framework's router) — the routing capability itself was
    confirmed, just the exact host to register.
  - The manifest callback **writes** the app-level credentials to a
    secret scope at runtime — the App's service principal needs **CAN
    WRITE/MANAGE** on that scope, not the platform's default READ-only
    declared permission, so this must be explicitly elevated. Also: the
    App's filesystem is ephemeral, so nothing (including these
    credentials) can be cached to local disk between restarts — always
    read from the secret scope. If the DAB schema doesn't support secret
    *values* as a declared resource, the bundle should only define the
    scope itself (if supported); actual GitHub private key/client secret
    values are always provisioned at first-run by the App backend, never
    stored in bundle YAML.
  - **Lakebase deploy-first schema ownership** (§1c): the App's service
    principal must deploy at least once before any local dev touches its
    Lakebase tables, since it creates and owns its Postgres schema on
    first deploy — local dev against the schema beforehand fails with a
    bare permissions error (`42501`), not a helpful message.
- Blocks every later phase that reads or writes skill files, so needs to
  exist before Phase 3 (Review App accept flow) or Phase 5 (Local
  Installer) can be exercised against anything other than this original
  repo. Doesn't block Phase 1/2/2b, which are workspace infra and don't
  care which skills repo is configured.
- **DAB scoping, cross-reviewed vs. `databricks-dabs`**: bundle
  variables/targets parameterize **deploy-time infra only** (catalog,
  schema, warehouse ID, volume/job/pipeline/app names, dev/prod
  workspace targets) — they are not the right place for the
  per-installation mutable settings this phase collects (GitHub
  `installation_id`, tracked cloud/region, target skills repo), which
  live in Lakebase (§1c) after deployment instead. Also: a Databricks
  App's own runtime environment variables belong in the app's own
  `app.yaml`, not in the bundle's `databricks.yml`, and a bundle deploy
  alone can leave the App resource stopped — deployment automation needs
  to explicitly run/start the App after `bundle deploy`, not assume it
  starts itself. Per §1d: the secret scope, Lakebase instance, and
  service principals this phase provisions belong to **`-infra`**; the
  App resource itself (and this "start it after deploy" step) belongs to
  **`-app`**, deployed last of the three.

### Phase 1 — Grounding corpus (docs ingestion)
- Scrape/sync the relevant `docs.databricks.com` sections into a UC Volume,
  chunk + embed into a **Delta Sync index with managed embeddings**
  (reuses `databricks-vector-search` patterns already staged in this
  repo). Pin the embedding endpoint to whatever's current per that skill
  at build time (e.g. `databricks-qwen3-embedding-0-6b` as of this
  writing) rather than a hardcoded/legacy model name — cross-reviewed
  against the vector-search skill, which has moved on from its older
  GTE/BGE examples.
- This corpus is *supplementary* context, not the primary source for a given
  announcement — the Research Agent always does a live fetch of the
  announcement's own links first (Phase 2). This corpus just gives it
  broader context for the surrounding docs area.
- **`ai_prep_search` considered — not adopted as designed (investigated,
  two independent lenses).** `ai_prep_search(parsed, [options])` is a real
  Databricks SQL/AI Function (Beta, DBR 18.2+, serverless env v3+,
  Python/SQL only) that turns **`ai_parse_document`'s output** into
  semantically-chunked, context-enriched, embedding-ready text
  (`chunk_to_embed`/`chunk_to_retrieve`), explicitly documented as the
  chunking stage ahead of a Vector Search (now "Databricks AI Search")
  Delta Sync index — i.e. it's meant to *replace* manual chunking.
  Its hard input contract is `ai_parse_document` output only (PDF, JPG/
  JPEG, PNG, TIFF, DOC/DOCX, PPT/PPTX; ≤500 pages/≤100MB) — other VARIANT
  shapes error or behave unexpectedly. This corpus is scraped
  `docs.databricks.com` **HTML pages**, not binary document files run
  through `ai_parse_document`, so `ai_prep_search` doesn't fit the
  ingestion path *as currently designed* without an artificial
  HTML→PDF conversion detour that isn't worth the complexity. Revisit
  only if this corpus's source ever shifts to genuine document files
  (e.g. ingesting downloadable Databricks whitepapers/PDF guides
  alongside/instead of scraped HTML) — at that point `ai_prep_search`
  is the right tool and should replace whatever ad hoc chunking Build
  Plan 02 lands on. Also confirmed **not** applicable to `research_log`
  (Phase 2/Build Plan 01) for the same reason — see that plan's own
  note. (One side finding worth recording: the locally-installed
  `databricks-ai-functions` skill doesn't mention `ai_prep_search` at
  all — a live, concrete instance of exactly the staleness gap this
  whole project exists to catch.)
- **DAB bundling notes (cross-reviewed vs. `databricks-dabs`)**, relevant
  to Phase 1 and Phase 2b alike: (a) UC Volume resources in the bundle
  need explicit **`grants`**/UC privileges (`READ_VOLUME`, and
  `WRITE_VOLUME` for writers) — not a generic bundle `permissions` block,
  which applies to job/app/pipeline resources, not volume access; (b)
  **Vector Search endpoints/indexes ARE declarable DAB resource types** —
  `resources.vector_search_endpoints` and `resources.vector_search_indexes`
  are present in the CLI v1.5.0 `databricks bundle schema` (confirmed by
  Build Plan 02's cross-review; this corrects an earlier "not confirmed"
  hedge here — itself an instance of exactly the staleness this project
  catches). **Declare them in the bundle.** A small idempotent
  **post-deploy setup job** remains a *contingency only*, for a specific
  runtime workspace limitation discovered at deploy time — not the default
  path.
  Per §1d: the UC Volumes and the pipelines/tables they feed belong to
  **`-infra`**; the Vector Search endpoint/index (and its post-deploy
  setup job, if needed) belongs to **`-ai-tools`**, deployed only after
  `-infra` has run at least once so the source Delta tables have rows.

### Phase 2 — RSS ingestion + Research Agent (the event pipeline)
- **RSS fetch is a scheduled Databricks Job, not the App** (investigated
  directly — plain RSS 2.0 has no push/webhook primitive; none of the 3
  feeds advertise a WebSub/PubSubHubbub `rel="hub"` link, confirmed live
  against all three, so a client *must* poll). The App is deliberately
  kept out of this loop: Databricks Apps have no durable in-process
  scheduler guarantee (status can go `Crashed`, local filesystem/in-memory
  state is lost on restart, and horizontal scaling means multiple app
  instances could double-fetch without an external lease) — Lakeflow Jobs'
  documented scheduled-trigger primitive is the right owner instead. The
  App stays focused on the review UI, run-status display, and a manual
  "run now" control that triggers the Job.
  - The Job fetches the installation's selected feed(s) (§1a, §6 #6 — any
    of AWS `docs.databricks.com/aws/en/feed.xml`, GCP
    `docs.databricks.com/gcp/en/feed.xml`, Azure `learn.microsoft.com/
    en-us/azure/databricks/feed.xml`, all public RSS 2.0, no auth) using
    **conditional GET** — send `If-None-Match` with the stored `ETag` from
    the last successful fetch; all three feeds confirmed to return a
    `304 Not Modified` (empty body) when unchanged, avoiding a ~1MB
    re-download (each feed is 800-900 items) on every no-op poll.
  - **Daily cadence is sufficient**: every `<pubDate>` across all three
    feeds is stamped `00:00:00 GMT` — Databricks publishes release notes
    per calendar day, not per clock time (confirmed: ~72% of the last 30
    days had at least one entry). Polling more often than daily buys no
    earlier signal; an afternoon/evening-UTC daily run aligns with when
    the feed's `<lastBuildDate>` typically updates.
  - Each `<item>`'s fields to extract: `<title>`, `<link>`, `<guid>`
    (identical to `<link>` in these feeds — **use this as the dedup key**,
    not title+link), `<pubDate>`, `<description>` (an HTML fragment inside
    CDATA, needs a second parse if plain text is wanted), and `<category>`
    (repeatable, ~1.9 per item on average — collect into a list, don't
    overwrite).
  - **No Zerobus** — RSS is pulled, not pushed, so there is no continuous
    producer needing a streaming-ingest client at all; Zerobus would add
    a gRPC stream, a dedicated publishing service principal, and
    generated protobuf schemas to solve a problem this pipeline doesn't
    have. The same notebook task that fetches the feed parses it and
    does a **plain batch append** straight into `rss_bronze` (raw and
    append-only) — no separate publish step, no separate compute path.
    A **silver table (`rss_silver`)** de-dupes on
    `<guid>` **within a bounded time window keyed off each entry's
    `pubDate`** — not an unbounded global keep-first, which would grow
    streaming dedup state indefinitely over the feed's already-900+-item
    history (cross-reviewed vs. `databricks-pipelines`' streaming-patterns
    guidance). Duplicate guid entries from a cross-cloud publish only ever
    arrive close together in time, so a bounded window is sufficient.
  - New-row arrival on `rss_silver` (i.e. post-dedup) is the trigger for
    the Research Agent, via a Databricks Jobs **Table update** trigger
    (`trigger.table_update`, `table_names: ["<catalog>.<schema>.rss_silver"]`,
    `condition: ANY_UPDATED`) — resolved §6 #13, confirmed against
    `databricks-jobs`. This is a real, UC-table-scoped native trigger type
    (distinct from `file_arrival`, which only covers Volumes/external
    locations, never managed tables), and is the sole trigger mechanism (an
    LDP flow cannot itself invoke a Job or agent as a side effect of a
    dataflow, only write to Streaming Tables/MVs/Sinks, so the earlier "or a
    pipeline step fires it" option was correctly dropped). **Important
    caveat**: the trigger fires on table **commit**, not per row — the
    Research Agent's Job must enumerate the actually-new rows itself using
    the trigger's exposed commit version/timestamp
    (`{{job.trigger.table_update...}}`) against an idempotent
    processed/seen marker, optionally via Delta Change Data Feed on
    `rss_silver` if efficient row-level change extraction is needed, rather
    than assuming "one trigger fire = one new row."
- **`rss_bronze`'s table design** — a plain UC managed Delta table (not a
  Lakeflow Declarative Pipeline STREAMING TABLE — DLT-managed table
  properties for these specific features aren't documented, and a plain
  managed table has no such gap), created once with every feature this
  project's storage design calls for set together at creation time:
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
  CDF, Row Tracking, a `VARIANT` column, and `CLUSTER BY AUTO` are all
  confirmed compatible together on one table (no documented conflict);
  Variant Shredding is Beta and needs a DBR floor + a workspace-admin
  Previews toggle enabled once per installation. **Assumption locked for
  this plan**: Variant Shredding is treated as available identically
  whether the table is written by a plain Job task (as `rss_bronze` is)
  or produced/consumed by a Spark Declarative Pipeline (as `rss_silver`
  and the findings-ingestion pipeline are, per Phase 2b) — i.e. these are
  full Delta table properties, not something gated by which engine wrote
  the table. This is taken as given rather than re-investigated; if a
  concrete DBR version pins this differently for pipeline-managed tables
  specifically, revisit then, not before.
  `rss_silver`'s Lakeflow Declarative Pipeline then reads `rss_bronze` via
  `STREAM(rss_bronze)` for its dedup transform — this is the actual
  streaming-table consumer in this design; `rss_bronze` itself only ever
  needs batch-append writes, never DLT-managed writes.
- **The daily polling notebook task**, in full (single Job task, runs
  once a day on serverless compute — no continuous producer, no gRPC
  stream, no dedicated publishing service principal needed since it's
  the Job's own identity doing a normal Delta write):
  1. **Load the cursor.** For each tracked feed (§1a's cloud selection —
     1 to 3 URLs), read that feed's stored `ETag`/`Last-Modified` pair
     from its Lakebase polling-cursor row (§1c). First-ever run per feed
     has no cursor — treat as an unconditional GET.
  2. **Conditional GET.** Request the feed URL with `If-None-Match:
     <stored ETag>` (and/or `If-Modified-Since: <stored Last-Modified>`
     as a fallback for servers that only honor one). A `304 Not
     Modified` response (confirmed live for all three feeds) means
     nothing changed since last run — skip straight to step 5 for this
     feed, no parsing needed. A `200 OK` means new content; capture the
     response's fresh `ETag`/`Last-Modified` headers for step 5.
  3. **Parse.** On `200`, parse the RSS 2.0 XML with a standard library
     (e.g. `feedparser` or `xml.etree`) into a list of item records,
     extracting `<title>`, `<link>`, `<guid>` (the dedup key — identical
     to `<link>` in these feeds), `<pubDate>`, `<description>` (HTML
     inside CDATA), and all repeated `<category>` values as a list.
     Stamp each record with which feed/cloud it came from and the
     current run's timestamp.
  4. **Append to bronze.** Build a small DataFrame (pandas → Spark, or
     Spark directly) from the parsed items and write it with
     `.write.format("delta").mode("append")` straight to `rss_bronze` —
     one plain batch write per run, not a per-item streaming call.
     `rss_bronze` must already exist with the schema + `TBLPROPERTIES`
     given above (created once, by the DAB or a setup step) — the raw
     item JSON lands in the `feed_value` VARIANT column as-is.
  5. **Save the cursor.** Whether this feed returned `200` (new
     `ETag`/`Last-Modified`) or `304` (headers unchanged, but still worth
     refreshing the "last checked" timestamp for observability), upsert
     that feed's row back into the Lakebase cursor table (§1c) so the
     next day's run starts from the right conditional-GET state.
  6. Repeat steps 1–5 independently per tracked feed within the same
     task run — a `304` on the AWS feed and a `200` on the Azure feed in
     the same run is the normal case, not an error path.

  This task needs no gRPC client, no protobuf schema generation, and no
  auth beyond the Job's own service-principal identity (which already has
  `MODIFY`+`SELECT` on `rss_bronze` and read/write on its Lakebase schema
  per §1c) — it is a plain HTTP-fetch-then-Delta-write, exactly the shape
  a scheduled Databricks Job notebook task is built for. Runs on
  **serverless compute** by default; nothing in this design has any
  compute-tier requirement, so there's no fallback-to-classic case to plan
  for here (that concern was specific to the Zerobus SDK's own egress
  behavior, which is no longer part of this pipeline).
- **Why this isn't a Lakeflow Declarative Pipeline (LDP) dataset using
  `http_request`, investigated directly** — considered and rejected on
  three independent grounds, not just style preference:
  1. **`http_request` can't even carry the cursor.** Its return type is
     `STRUCT<status_code INT, text STRING>` only — no response headers
     are exposed, so `ETag`/`Last-Modified` from a `200` response can
     never be read back to save for next time. Conditional GET is
     impossible with this function regardless of where it runs.
  2. **It's explicitly scoped away from this workload.** Per its own
     docs: *"designed for interactive and agent-based use cases, not for
     high-volume batch queries... For new code, Databricks recommends
     using the Unity Catalog connections proxy endpoint with the
     provider's SDK instead."* It also requires a pre-created UC HTTP
     `CONNECTION` object (admin-provisioned, `USE CONNECTION` grant) as
     a new piece of per-installation setup this design doesn't otherwise
     need.
  3. **LDP's execution model actively fights a side-effecting HTTP call
     inside a dataset definition, independent of which function makes
     the call.** No documented LDP source type is HTTP-based (Auto
     Loader/Kafka/Kinesis/Pub-Sub/Pulsar/Event Hubs/JDBC/Delta/custom
     data source only — no REST/HTTP entry). Pipelines retry at the
     task, flow, *and* pipeline level on transient failure, and a full
     refresh re-runs the whole dataset — either can silently re-issue
     the same "fetch" multiple times with no exactly-once guarantee.
     Worse, saving the new cursor back is a read-*then-write* of
     external state, which the declarative dataset model doesn't
     support mid-flow (a dataset function returns a DataFrame; the only
     sanctioned external-write path, a `Sink`, is append-only streaming,
     not a point read-modify-write of one cursor row). A `TRIGGERED`
     pipeline also has no native schedule of its own either way — "daily"
     still has to come from a Jobs schedule wrapping it.

  A plain Job task has none of these problems: it reads status code
  *and* headers from any HTTP client, freely does imperative
  read-cursor → conditional-GET → append → write-cursor once per run,
  and needs no UC Connection object. LDP stays doing exactly what it's
  built for — the downstream `rss_bronze` → `rss_silver` dedup transform.
- **Research Agent** (one run per new `rss_silver` row): per the
  architecture diagram above. Key design points:
  - Always does a LIVE fetch of the announcement body + every link it
    references — never relies solely on the Phase 1 corpus, since
    same-day announcements often link to pages not yet indexed.
  - Citations are first-class output, not an afterthought: every claim in
    the summary should be traceable to a specific fetched URL. Use
    **`ai_query`**, not `ai_summarize`, for the synthesis step — this
    needs cross-document reasoning over multiple fetched sources with a
    citation contract, which is what `ai_query` is documented for;
    `ai_summarize` is single-document condensation with no citation
    structure (cross-reviewed vs. `databricks-ai-functions`).
  - `ai_classify` (**multilabel**) needs a maintained taxonomy — the label
    set is "all currently installed/staged skill names + one-line
    descriptions," PLUS an always-present **"New Feature"** bucket for
    announcements that don't fit any existing skill well. This taxonomy
    must be kept in sync with the skill directory (regenerate it from
    `SKILL.md` frontmatter each run) and with any user-created buckets
    (see Phase 3's override flow). **Guardrails** (cross-reviewed vs.
    `databricks-ai-functions`' documented limits): keep labels ≤100 chars
    and descriptions ≤1000 chars each, cap the taxonomy well under the
    500-label ceiling (merge/archive stale user-created buckets before it
    gets there), and pass any classification context via the function's
    `instructions` option rather than folding unbounded taxonomy text
    into the label list itself. `ai_classify` does **not** natively return
    a confidence score or rationale — the plan's earlier "ranked list with
    confidence + rationale" wording overstated what it returns; if the
    review UI wants those, compute them with a small separate LLM step,
    not as `ai_classify` output.
  - Batch AI Function calls (this step and step 3/5) use **non-throwing,
    error-capturing execution** (e.g. `failOnError => false` where
    available) rather than aborting the whole run on one bad call — a
    failed classification/extraction/summary for one entry writes an
    error payload to a dead-letter/`_errors` table and triggers a Phase 7
    alert, instead of silently dropping the RSS entry or crashing the Job.
  - Output is always a *proposal* written to a JSON findings file (never a
    direct write to a skill file or straight to Delta — see Phase 2b).

### Phase 2b — Ingestion pipeline (findings → queryable + searchable)
- A Lakeflow Declarative Pipeline (streaming, Autoloader) watches the JSON
  findings volume the Research Agent writes to and loads it: bronze (raw
  JSON) → silver (`research_log` Delta table, one row per skill
  classification per RSS entry). **Correction after resolving a conflict
  between two cross-reviews**: the LDP only ever writes normal rows to
  `research_log` — it does **not** use a `ForEachBatch Sink` to manually
  call the Vector Search API (an earlier draft of this plan said that;
  that pattern is for Direct Access/manual-CRUD indexes, not this design).
  A separate **Delta Sync index (managed embeddings) with
  `pipeline_type: TRIGGERED`** is declared on `research_log`; because
  `TRIGGERED` syncs on-demand rather than automatically on every table
  change, the pipeline's final step calls the Vector Search
  `sync_index()` API after each batch lands in `research_log`, rather
  than relying on it to happen for free.
- This keeps the Research Agent task itself simple and idempotent (just
  write a file); all the "land it in tables and make it searchable" logic
  lives in one reusable streaming pipeline instead of being duplicated
  across every place that produces findings.

### Phase 3 — Review App (the human gate + editor)
- Databricks App (AppKit + React) reading `research_log` + the skill
  directory. **Reading `research_log` goes through the Apps Data Access
  Decision Gate** (cross-reviewed vs. `databricks-apps`): default to
  warehouse **Analytics** (`config/queries/` + `useAnalyticsQuery`), not a
  custom endpoint running a raw `SELECT` — that pattern is explicitly
  discouraged. Only reach for a **Lakebase synced table** (read-only
  mirror of `research_log`, §1c) if the Analytics path proves too slow
  for the review queue in practice.
- Review queue UI: one card per pending research entry, with per-skill
  accept/reject, full diff view, and a markdown editor for hand-editing the
  proposed text before accepting. **The accept/reject/edit-then-accept
  click-state itself is CRUD, and lives in Lakebase (§1c)** — it is never
  written back into the Delta `research_log` table, which stays an
  append-oriented record fed only by Phase 2b's pipeline.
- Markdown viewer/editor works on *any* skill file at any time (not just
  ones with a pending proposal) — view current, edit, save back to the
  configured skills repo (§1a) as a PR.
- Accept action forks into: (a) commit + PR to the configured skills repo
  (always), and optionally (b) trigger the Local Installer for the current
  session's machine.
- **Classification override**: every classified skill/bucket on a
  `research_log` entry is user-editable in the review card — the user can
  reassign it to a different existing skill, to "New Feature," or type in a
  brand-new bucket name that doesn't exist yet in the taxonomy. Any override
  or new-bucket creation re-triggers the skill-update generation step (Phase
  2 step 5) for that entry under the corrected label, rather than just
  relabeling the old proposal — so the proposed diff is always grounded in
  the label the human actually confirmed, not the model's first guess.
  **This re-trigger — and any other App→Job call — uses AppKit's `jobs()`
  plugin** (cross-reviewed vs. `databricks-apps`), never a bespoke
  Jobs-SDK custom endpoint, and is always **fire-and-poll** (`runNow` then
  poll run status), never a synchronous run-and-wait: the Apps platform's
  reverse proxy enforces a hard **120-second per-request timeout**, well
  under what a full research/classify/extract/diff regeneration run could
  take.
  A new bucket the user creates becomes a permanent taxonomy entry (and,
  eventually, a new skill directory) for future classification runs.
- **Starter-pack check (§1a)**: the same review queue also accepts entries
  from an on-demand "check for latest from `<starter pack>`" action (not
  just RSS-driven proposals) — a per-skill or full-sweep diff against the
  pack's source repo, reviewed and accepted/rejected exactly like a
  research-generated proposal. No separate UI needed; it's the same queue,
  same diff view, same edit-then-accept path, just a different origin for
  the proposed content.

### Phase 4 — Version tracking + CLI-drift detection
- **Versioning scheme** (see §5 for the concrete format): every skill file
  this project manages carries `base_cli_version` (the `aitools`/CLI version
  it started from) and its own independently-incrementing
  `updated_version` / `updated_at`. This never touches or overwrites the
  CLI's own version string. **The queryable version-tracking table this
  phase's three-way comparison reads/writes lives in Lakebase (§1c)**, not
  Delta — the `SKILL.md` frontmatter in the repo stays the authoritative
  source of record, and the Lakebase table is its interactively-queried
  projection (cross-reviewed vs. `databricks-lakebase`).
- **Simplified by the CLI investigation**: since skill content is fetched
  by the CLI from the public `databricks/databricks-agent-skills` GitHub
  repo (not bundled in the binary), and `internal/build/cli-compat.json`
  maps CLI versions → compatible skills-repo refs, this project does **not**
  need to install a new CLI version to see what it ships. On every new CLI
  release: fetch that repo's `cli-compat.json` mapping (or the CLI's own
  changelog) to resolve the new CLI version → skills-repo ref, then fetch
  that ref's `manifest.json` + skill files directly via the same
  `raw.githubusercontent.com` pattern the CLI itself uses. Compare the
  fetched content + its `SKILL.md` `metadata.version` against the
  corresponding row's `base_cli_version` + `updated_version` in this
  project's tracking table. Three outcomes per skill:
  1. CLI's shipped content is newer/equal → surface "CLI has caught up,
     consider re-basing your local copy onto the CLI's version instead."
  2. This project's `updated_version` is still ahead → no action needed,
     flag informationally.
  3. Both have diverged independently since the shared `base_cli_version` →
     flag as a **conflict requiring human reconciliation** (three-way diff
     in the App), don't silently pick a winner.

### Phase 5 — Local Installer
- **Resolved (was: reverse-engineer `aitools install`'s per-agent
  placement)** — investigation confirmed `aitools install` cannot be
  piggybacked: it hardcodes its source to
  `github.com/databricks/databricks-agent-skills` (no repo/ref override
  flag), so it can never be reused as-is even for the agents it does
  support.
- **Scope for now**: target exactly the CLI's own supported agent set —
  Claude Code, Cursor, Codex CLI, OpenCode, GitHub Copilot, Antigravity. Pi
  is explicitly out of scope for this phase (no CLI-side convention exists
  and it's not a priority right now); revisit later if needed.
- **Primary mechanism**: git-based. Phase 3's accept flow already commits
  approved changes as a PR to the configured skills repo (§1a, "always") —
  extend that with a small, reviewed-once `apply-skill.sh` checked into
  that same repo (present from the starter pack, or added at first setup
  for a from-scratch repo) that the user runs after `git pull`. The script
  detects which of the above local agents are present and fans the
  approved skill out to each one's own directory/format. Security profile:
  the payload was already reviewed as a PR diff; the only local execution
  is a small, stable, auditable script rather than a freshly generated
  blob per action.
- **Concrete per-agent raw-skill paths** (confirmed from CLI source, useful
  reference even though we're not calling the CLI itself): Claude Code
  `~/.claude/skills` (global) / `<cwd>/.claude/skills` (project); Cursor
  `~/.cursor/skills`; Codex `~/.codex/skills`; GitHub Copilot
  `~/.copilot/skills`; OpenCode `${XDG_CONFIG_HOME:-~/.config}/opencode/
  skills` (or `$APPDATA/opencode/skills` on Windows). This is the full
  agent set this phase targets — Pi is out of scope for now (no CLI-side
  convention exists, and it's not a current priority; revisit later).
- **New risk to design around**: the CLI's current default is
  **plugin-first** for Claude Code/Codex/Copilot (each uses that agent's
  own plugin-install mechanism; only Cursor/OpenCode/Antigravity get raw
  files by default). If a user's Claude Code/Codex setup is plugin-managed,
  writing raw files into `~/.claude/skills/` directly could be silently
  overwritten or conflict on the next `aitools install`/plugin update.
  `apply-skill.sh` should detect plugin-vs-raw mode per agent (e.g. check
  for the CLI's own `.state.json`/plugin records) before deciding how to
  apply, rather than always writing raw files blindly.
- **Fallback mechanism**: for users without the configured repo cloned,
  the App also offers a downloadable one-off script from a short-TTL signed URL —
  scoped to file-writes only (no remote `eval`), and shown to the user
  before it's piped into a shell, not silently `curl | bash`'d.
- **Rejected**: a local companion daemon (ongoing build/sign/distribute
  cost disproportionate to a low-frequency file-copy action, and doesn't
  actually remove the arbitrary-code-execution problem — just relocates it
  to the daemon's own install step).
- Every file the installer writes gets the version-tag frontmatter from §5
  so a later drift check can always identify provenance.
- Optional convenience add-on, not primary: a Chromium-only "grant folder
  access" button via the browser File System Access API, for users who
  don't want to touch a script or git at all.

### Phase 6 — Knowledge base search + MCP server (stretch)
- Expose `research_log` + the Vector Search index as: (a) a search UI in
  the App, and (b) a Databricks App-hosted MCP server so other agents can
  query it programmatically (e.g. "what's the latest guidance on X",
  "which skill covers Y").
- **Two separate indexes is deliberate, not an oversight** (cross-reviewed
  vs. `databricks-vector-search`): the Phase 1 docs-corpus index and the
  Phase 2b `research_log` index have different source Delta tables,
  different schemas/`columns_to_sync`, and different refresh cadences
  (periodic vs. per-announcement) — a Delta Sync index maps 1:1 to one
  source table, so merging them isn't the right move. Both can share one
  Storage-Optimized Vector Search endpoint to avoid provisioning two.
- Default `research_log` search/MCP queries to **HYBRID** (semantic +
  keyword/BM25), not pure ANN — these queries frequently contain exact
  skill names, feature names, and compliance terms (HIPAA, PCI-DSS, etc.)
  that need literal matching alongside semantic recall.
- This is additive — build after Phases 1–5 are solid, since it's a new
  consumer of the same data rather than a dependency of the core loop.

### Phase 7 — Guardrails & observability
- Failure alerting to Slack (RSS fetch failure, LLM call failure, GitHub
  API failure, ai_classify low-confidence-for-all-skills case) so a bad run
  doesn't silently drop an announcement.
- Every proposed change ships as a PR and/or a locally-reviewed apply —
  never an unattended auto-merge or auto-overwrite.

## 3. Skills this reuses

| Phase | Relevant staged skill |
|-------|------------------------|
| Docs corpus + retrieval | `databricks-vector-search`, `databricks-unity-catalog` (volumes) |
| RSS watcher + Research Agent | `databricks-jobs`, `databricks-ai-functions` (`ai_classify`, `ai_extract`, `ai_query`) |
| Per-installation OLTP state (Phase 0, Phase 3, §1c) | `databricks-lakebase` |
| Review App | `databricks-apps`, `databricks-apps-python`, `databricks-app-design`, `databricks-lakebase` |
| Knowledge base / MCP server | `databricks-vector-search`, `databricks-model-serving` (if fronting via an endpoint) |
| Bundling everything | `databricks-dabs` (ship as **three** dependency-ordered bundles per §1d — `-infra`: schemas/volumes/secret scopes/Lakebase/service principals/target tables/pipelines/the RSS Job; `-ai-tools`: Vector Search endpoints/indexes, Genie Spaces, UC-registered MCP tools; `-app`: the App itself — not one "Job + App" bundle) |

## 4. Suggested build order

0. Phase 0 (workspace setup / repo configuration) is a prerequisite for
   this project's *own* development too, not just for future installers —
   even developing against this repo should exercise the same
   configuration path (pointed at this repo) rather than hardcoding it, so
   Phase 0's plumbing is proven from day one instead of retrofitted later.
   This is also where the three-bundle deploy order (§1d) first gets
   exercised for real: `-infra` deploys and its Job/pipelines run at
   least once, *then* `-ai-tools`, *then* `-app` — proving the sequencing
   script itself, not just each bundle's YAML in isolation.
1. Phase 2 (RSS watcher + Research Agent), run manually against one
   real RSS entry end-to-end, to prove the fetch → summarize → cite →
   classify → propose loop works before investing in a UI.
2. Phase 1 (real grounding corpus) alongside/soon after — the Research
   Agent needs *some* corpus from day one, but perfect coverage isn't
   required to prove Phase 2.
3. Phase 3 (Review App) once there's a handful of real `research_log` rows
   to review.
4. Phase 5 (Local Installer) once a human has approved at least one
   proposal in the App and wants to actually apply it.
5. Phase 4 (CLI-drift detection) once the version-tag scheme (§5) is in use
   long enough that there's something to compare against a new CLI release.
6. Phase 7 (schedule + alerting) once the pipeline runs unattended.
7. Phase 6 (search + MCP server) last — pure upside, no dependents.
8. The "starter pack" registry + multi-pack support (beyond the single
   "Matt's version" option) is deliberately last — one working pack is
   enough to prove Phase 0 end-to-end; more packs are additive.

## 5. Versioning scheme (proposed, revised after CLI investigation)

The `databricks-agent-skills` repo the CLI fetches from already puts a
`metadata.version` field in each `SKILL.md`'s YAML frontmatter (confirmed at
`skills/databricks-core/SKILL.md:1`). Reuse that same key/location for
consistency with the upstream format, and add this project's own fields
alongside it rather than inventing a parallel scheme:

```yaml
metadata:
  version: "1.5.0+au.3"        # this project's own increment; base semver segment mirrors upstream's metadata.version format
base_cli_version: "1.5.0"       # aitools/CLI version this skill's content started from
base_skills_repo_ref: "v1.5.0" # the databricks-agent-skills ref that base_cli_version resolved to, via cli-compat.json
updated_at: "2026-07-05T00:00:00Z"
last_research_log_id: "rl_00042"
availability:                   # see §1b — cloud + region-scoped compliance support, extracted by the Research Agent
  clouds: [aws, azure, gcp]
  compliance:                    # keyed by this installation's tracked regions (§1a)
    no_csp:
      us-west-2: true
    hipaa:
      us-west-2: true
    pci_dss:
      us-west-2: false
    fedramp_moderate:
      us-west-2: unknown
```

- `base_cli_version` + `base_skills_repo_ref` are set once, at the point a
  skill first gets tracked (or re-based per Phase 4's outcome 1). Storing
  the resolved skills-repo ref (not just the CLI version) means Phase 4
  never has to re-resolve the CLI-version-→-ref mapping for a historical
  comparison.
- `metadata.version` increments every time this project's pipeline (or a
  human edit in the App) changes the file — independent of CLI versioning,
  so the CLI's own release numbering is never touched or implied to be
  something it isn't.
- This is what makes the three-way Phase-4 comparison possible: the
  upstream repo's `metadata.version` at the new CLI's resolved ref vs. this
  file's `base_cli_version`/`base_skills_repo_ref` vs. its own
  `metadata.version` tells you exactly who has moved since the shared
  starting point.
- The App-queryable copy of this tracking data lives in **Lakebase**
  (§1c, §2 Phase 4) — the frontmatter above, in the repo, remains the
  single source of truth; Lakebase just gives the App fast, interactive
  read/write access to it for the three-way comparison UI.

## 6. Open questions for the repo owner

1. ~~Docs freshness for brand-new announcements~~ — **Resolved**: yes, the
   Research Agent always does a live fetch of an announcement's linked URLs;
   the Vector Search corpus is supplementary context only. Reflected in
   Phase 2 above.
2. ~~Local Installer reach~~ — **Resolved**: git-based PR + local
   `apply-skill.sh` (reviewed-once, checked into this repo) is the primary
   mechanism; a downloadable, scoped, review-before-run script is the
   fallback for users without the repo cloned. A local companion daemon was
   considered and rejected. Reflected in Phase 5 above. Confirmed along the
   way: `databricks aitools install` cannot be piggybacked (no
   source-override flag; `--agents` doesn't include Pi), so this project
   needs its own installer regardless.
3. ~~`aitools install` internals~~ — **Resolved.** Source: `databricks/cli`
   (`cmd/cmd.go:108`). Key facts, all confirmed against source:
   - Skill **content is not bundled in the CLI binary** — it's fetched over
     HTTPS from `raw.githubusercontent.com/databricks/databricks-agent-
     skills/<ref>/...` (`source.go:23`, `installer.go:178/221`), driven by
     that repo's `manifest.json`. This means Phase 4's CLI-drift check can
     just fetch that public repo directly — **no need to install new CLI
     versions at all** to see what a given release ships. The only
     embedded artifact is `internal/build/cli-compat.json` (`go:embed`),
     which maps CLI versions → compatible Agent Skills repo refs
     (`internal/build/clicompat.go:5`) — exactly the lookup Phase 4 needs
     to resolve "CLI version X ships skills-repo ref Y."
   - Agent detection is hardcoded to 6 agents (Claude Code, Cursor, Codex
     CLI, OpenCode, GitHub Copilot, Antigravity) via config-dir existence
     and `PATH` lookup (`libs/aitools/agents/agents.go:149`, `detect.go:43`)
     — **confirms Pi has no CLI-side detection at all**, consistent with
     the local-installer finding above.
   - Default behavior on current `main` is **plugin-first** for
     Claude/Codex/Copilot (uses each agent's own plugin-install mechanism);
     only Cursor/OpenCode/Antigravity get raw skill files by default, and
     `--skills-only` forces raw files for all agents (`install.go:64/293`).
     **New risk flagged**: if a user's Claude Code/Codex install is
     plugin-managed, this project's raw-file `apply-skill.sh` (Phase 5)
     writing directly into `~/.claude/skills/` could be silently
     overwritten or fought over on the next `aitools install`/plugin
     update — needs a design decision (detect plugin-vs-raw mode per agent
     before writing, most likely).
   - Canonical local storage matches this project's existing assumption:
     `~/.databricks/aitools/skills/<skill>` (global) or
     `<cwd>/.databricks/aitools/skills/<skill>` (project scope)
     (`state.go:155/164`), with a `.state.json` tracking `release`,
     `last_updated`, per-skill versions, and per-file SHA256 provenance
     (`state.go:28/40`). Each skill's own `SKILL.md` already carries a
     `metadata.version` front-matter field (e.g. `skills/databricks-core/
     SKILL.md:1`) — **this project's versioning scheme (§5) should reuse
     that same front-matter key/location** rather than inventing a
     parallel one, for consistency with the upstream format.
   - No stable external Go API for programmatic reuse (internal packages
     like `installer.InstallSkillsForAgents` exist but aren't a supported
     surface) — confirms Phase 5 must ship its own installer logic rather
     than importing the CLI's.
   - `aitools` (and this whole mechanism) shipped in CLI `v1.5.0`
     (`CHANGELOG.md:25/139`).
4. ~~Upstream fix path~~ — **Resolved, scope narrowed**: the source-of-truth
   repo is confirmed public (`github.com/databricks/databricks-agent-
   skills`), and **per repo owner, this is an owner-only capability, not a
   general product feature** — the app will never open upstream PRs on
   behalf of any other installation/user, only (potentially) for the repo
   owner's own installation. This substantially narrows what §2's earlier
   "what's needed" breakdown requires:
   - **No per-user opt-in UI, no per-user auth flow.** Drop the "Also
     propose upstream" review-card action and the fork-based per-user
     OAuth flow entirely from the general product surface — those were
     only needed if every installation could trigger this.
   - **Auth is a single, fixed identity**: the owner's own GitHub account
     (personal OAuth token or PAT, used only for this one maintainer-only
     flow), completely separate from Phase 0's per-installation GitHub
     App, and never distributed to other installations.
   - **Gated by an identity/role check** on the installation itself — the
     upstream-PR action (if built) should only render/execute when the
     running installation is the owner's own, not exposed as a toggle any
     other user could enable.
   - **Still needs the content-translation step and the investigation into
     `databricks-agent-skills`'s contribution policy/schema** from §2's
     breakdown — those don't go away, they're just now scoped to a single
     fixed contributor identity instead of N users' identities.
   - **Still a nice-to-have, lowest priority** — build after Phases 1–5,
     as an admin/maintainer-only tool separate from the mainline
     multi-tenant Phase 0–5 product surface, not bundled into it.
5. ~~New-skill detection~~ — **Resolved**: `ai_classify`'s taxonomy always
   includes an explicit **"New Feature"** bucket for announcements that
   don't fit an existing skill well, and the App lets a user override any
   classification (including typing in a brand-new bucket name). Any
   override or new-bucket creation re-triggers skill-update generation for
   that entry. Reflected in Phase 2 and Phase 3 above.
6. ~~RSS feed identity~~ — **Resolved.** Databricks publishes an official,
   public **RSS 2.0** feed per cloud (not per product area — SQL/ML/
   Workflows/etc. are all combined in one feed per cloud), each linked
   directly from its own "Release notes" docs page:
   - AWS: `https://docs.databricks.com/aws/en/feed.xml` (~900 items, back
     to Jan 2025 — richest history, canonical docs domain)
   - GCP: `https://docs.databricks.com/gcp/en/feed.xml` (~819 items)
   - Azure: `https://learn.microsoft.com/en-us/azure/databricks/feed.xml`
     (~882 items — hosted on Microsoft Learn, not docs.databricks.com;
     there is no `docs.databricks.com/azure/en/feed.xml`)
   - All three are fully public (no auth), verified live (HTTP 200,
     `application/xml`/`text/xml`), generated by the docs site's own
     RSS generator (`<generator>rssFeed.ts</generator>`) — plain RSS 2.0,
     no extra namespaces, no advertised push/WebSub hub. Each `<item>` has
     `<title>`, `<link>`, `<guid>` (identical to `<link>` in these
     feeds — the dedup key), a day-granularity `<pubDate>` (always
     `00:00:00 GMT`), an HTML `<description>`, and 0..N `<category>` tags.
     Both `ETag` and `Last-Modified` are honored for conditional GET
     (confirmed `304` on repeat fetch) — see Phase 2 for the polling
     mechanics this enables.
   - **Decision (revised): user-selectable per installation, not fixed**
     (§1a). Each installation picks any combination of the three feeds to
     track, defaulting to the cloud it's deployed on (auto-detection
     mechanism: see new open question #10 below). All three selectable at
     once remains supported. Content overlaps heavily across clouds, so
     whenever 2+ feeds are selected, dedup on **`<guid>`** happens in a
     **silver table (`rss_silver`)**, not at bronze
     — `rss_bronze` stays raw and append-only (exactly what was received,
     no dedup logic at that layer), and the Research Agent trigger keys
     off new `rss_silver` rows instead. Otherwise a cross-cloud
     announcement would fire the pipeline more than once for the same
     underlying change. Phase 2's RSS ingestion is therefore a **fan-out of
     1-3 producers (per the installation's selection) into one raw
     `rss_bronze` table, deduped into `rss_silver`**.
7. ~~Pi's local skill directory convention~~ — **Dropped for now, per repo
   owner**: Phase 5's Local Installer targets exactly the CLI's own
   supported agent set (Claude Code, Cursor, Codex CLI, OpenCode, GitHub
   Copilot, Antigravity). Pi is out of scope until there's a reason to
   revisit it — no invented convention needed today.
8. ~~GitHub connection mechanism for §1a~~ — **Resolved: GitHub App**,
   fully provisioned through the Databricks App's own UI — no manual
   GitHub settings-page form-filling, no hand-copied client secrets or
   private keys, no separate CLI steps. Concretely: a one-time admin setup
   screen drives GitHub's App Manifest flow to register the app itself and
   store its credentials as Databricks secrets; a per-installation "Connect
   GitHub repo" button drives GitHub's App installation picker and stores
   the resulting `installation_id`. Installation access tokens are minted
   on-demand server-side after that — no PAT is ever generated or handled
   by a user. The one unavoidable exception: GitHub requires the human's
   consent click to land on `github.com` itself for both flows (by design,
   not something an embedding app can remove) — everything else stays
   inside the Databricks App. Reflected in §1a and Phase 0 above.
9. ~~Starter pack as ongoing reference, not just one-time seed~~ —
   **Resolved**: yes — a starter pack stays a live, on-demand reference
   source. The user can trigger "check for latest from `<starter pack>`"
   (per-skill or full sweep) at any time, which diffs against the pack's
   current source content and feeds the result into the same review queue
   as RSS-driven proposals, with the same accept/reject/edit-then-accept
   options. Reflected in §1a and Phase 3 above.
10. ~~Auto-detecting the deployment cloud~~ — **Resolved.** Use the
    Databricks Python SDK: `WorkspaceClient().config.cloud` (falling back
    to `.config.environment.cloud`), which returns `"AWS"`, `"AZURE"`, or
    `"GCP"` directly — no REST call or account-level credentials needed.
    Databricks Apps automatically get `DATABRICKS_HOST` + service-principal
    credentials in their environment, so this works out of the box for a
    Databricks App (confirmed: Apps don't get a default `DATABRICKS_CLOUD`
    env var, but the SDK's `config.cloud` resolves it anyway). Fallback if
    ever needed without the SDK: infer from the `DATABRICKS_HOST` hostname
    suffix (`*.azuredatabricks.net`/`.databricks.azure.us`/`.azure.cn` →
    Azure; `*.gcp.databricks.com` → GCP; `*.cloud.databricks.com`/
    `.cloud.databricks.us` → AWS). Avoid the Account Workspaces API (needs
    account-level creds, not appropriate from inside an app) and avoid
    `dbutils`/Spark-internal config (Apps aren't a notebook Spark driver by
    default). This is Phase 0's concrete mechanism for the cloud-tracking
    default in §1a.
11. ~~Compliance-availability docs source~~ — **Resolved.** Yes — better
    than expected: Databricks publishes a structured **"Regional support
    for features" HTML table per compliance standard** (Feature × Region
    grid) on each standard's own docs page (HIPAA, PCI-DSS, FedRAMP
    Moderate, plus IRAP/HITRUST/ISMAP/TISAX/C5/K-FSI), and a cross-feature
    preview/beta support table on the main CSP page. Server-rendered HTML,
    not a JSON/API feed, but consistent and scrapable. Confirmed **FedRAMP
    is AWS-only** (no GCP/Azure FedRAMP page exists). Individual
    feature/release-notes pages sometimes carry their own compliance notes
    too, but that coverage is inconsistent — the three standard pages are
    the authoritative primary source. Exact URLs and the extraction
    approach are now in §1b above.
12. ~~Auto-detecting the deployment region~~ — **Resolved: no reliable
    universal method exists**, unlike cloud (#10). The Databricks SDK's
    `WorkspaceClient` exposes no workspace-scoped region field; the only
    authoritative programmatic source is the **account-level** Workspaces
    API (AWS: `aws_region`; GCP: `location`) or Azure Resource Manager's
    `Microsoft.Databricks/workspaces` `location` field — both need
    account/management-plane credentials a Databricks App doesn't get by
    default. Hostname inference is explicitly unreliable for modern
    per-workspace URLs (confirmed — legacy region-encoded hostnames still
    exist but Databricks itself advises against relying on them). No
    default env var is exposed to Apps either. **Decision**: best-effort
    account-API check if those credentials happen to be configured,
    otherwise the user picks tracked region(s) explicitly at setup with no
    pre-filled default. Reflected in §1a and Phase 0 above.
13. ~~Jobs' native table-arrival trigger support~~ — **Resolved.** Yes —
    Databricks Jobs has a native **Table update** trigger
    (`trigger.table_update`) scoped to UC managed Delta/Iceberg tables
    (also materialized views, streaming tables, some UC views), configured
    with `table_names` + `condition: ANY_UPDATED`/`ALL_UPDATED` and
    optional debounce (`min_time_between_triggers_seconds`,
    `wait_after_last_change_seconds`). This is a genuinely different
    trigger from `file_arrival`, which is scoped only to UC external
    locations/Volumes and explicitly excludes managed tables — the plan's
    earlier "file/table-arrival" phrasing conflated the two. One real
    constraint to design around: the trigger fires on table **commit**,
    not per new row, so the Research Agent's Job must still enumerate
    which rows are actually new via the trigger's exposed commit
    version/timestamp against an idempotent processed-marker (optionally
    using Delta Change Data Feed for efficient row-level extraction).
    Reflected in §1/Phase 2 above.
