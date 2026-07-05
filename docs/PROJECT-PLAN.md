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
│  - RSS feed entries are published to Zerobus, landing in a bronze   │
│    Delta table (`rss_bronze`) as they arrive                         │
│  - Arrival of a new `rss_bronze` row is itself the trigger — either  │
│    a file/table-arrival-triggered Job run, or a Lakeflow Declarative │
│    Pipeline flow that fires the Research Agent task per new row      │
│  - Falls back to a scheduled poll of the RSS feed only if Zerobus    │
│    ingestion isn't available end-to-end (see open question below)    │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ Research Agent (Job task, one run per new rss_bronze row)           │
│  1. LIVE fetch of the announcement body + every link it references — │
│     never wait on the next scheduled corpus refresh, since same-day  │
│     announcements often link to pages not yet indexed                │
│  2. Pull current docs.databricks.com context for the subject         │
│     (Vector Search index) as supplementary grounding                  │
│  3. Summarize what changed, in plain language, with inline            │
│     footnote-style citations back to every source URL used            │
│  4. ai_classify the announcement against the taxonomy of installed    │
│     skills (skill name + description = the label set, PLUS an         │
│     always-present "New Feature" bucket for anything that doesn't      │
│     fit an existing skill) to get a ranked list of affected             │
│     skill(s)/bucket, each with a confidence score + rationale           │
│  5. For each affected skill (or "New Feature" candidate): draft a       │
│     proposed SKILL.md/references patch (unified diff) grounded in       │
│     the summary + citations                                             │
│  6. Write findings to a JSON file (one per rss entry) in a UC Volume —  │
│     summary, citations[], skill_classifications[], proposed_diffs[],    │
│     status ("pending_review") — designed to be auto-loaded by a          │
│     Lakeflow Declarative Pipeline (streaming, Autoloader-based) into     │
│     `research_log` and the Vector Search index, rather than written      │
│     directly to Delta from the agent itself                              │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ Ingestion Pipeline (Lakeflow Declarative Pipeline, streaming)       │
│  - Autoloader watches the JSON findings volume → bronze → silver     │
│    (`research_log` Delta table) → embeds proposed-diff text and       │
│    updates the Vector Search index                                    │
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
     installation's config (a small state table, not a secret — the ID
     alone isn't a credential) and mints short-lived **installation access
     tokens** on-demand server-side whenever it needs to commit or open a
     PR. No PAT, and no long-lived credential the user has to generate or
     paste themselves.
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
- This has no effect on Phase 4's CLI-drift logic (still comparing against
  the public `databricks-agent-skills` repo, independent of which skills
  repo this installation is configured to use) or on the RSS/Research
  Agent pipeline (workspace infra, not repo-specific) — it only changes
  *which repo* Phase 3's accept flow and Phase 5's Local Installer target.

## 2. Build phases

### Phase 0 — Workspace setup / repo configuration
- First-run onboarding flow per §1a: connect a GitHub repo via a **GitHub
  App**, choose "start from scratch" vs. "seed from a starter pack" (with
  "Matt's version" as the first available pack), and store the resolved
  repo + installation config.
- Two-part GitHub App provisioning (resolved, §1a and §6 #8), both build
  tasks for this phase:
  1. **App-level, one-time**: an admin setup screen implementing GitHub's
     App Manifest flow (generate manifest → redirect to GitHub to confirm
     → callback exchanges the temporary code for app ID, private key,
     webhook secret, client ID/secret → store as Databricks secrets). Only
     run once per deployment of this project, not per installation.
  2. **Per-installation**: a "Connect GitHub repo" button that redirects to
     GitHub's App installation picker and back with an `installation_id`,
     stored in a small per-installation state table. Installation access
     tokens are minted on-demand server-side from then on — no PAT ever
     stored or handled.
- Blocks every later phase that reads or writes skill files, so needs to
  exist before Phase 3 (Review App accept flow) or Phase 5 (Local
  Installer) can be exercised against anything other than this original
  repo. Doesn't block Phase 1/2/2b, which are workspace infra and don't
  care which skills repo is configured.

### Phase 1 — Grounding corpus (docs ingestion)
- Scrape/sync the relevant `docs.databricks.com` sections into a UC Volume,
  chunk + embed into a Vector Search index (reuses `databricks-vector-search`
  patterns already staged in this repo).
- This corpus is *supplementary* context, not the primary source for a given
  announcement — the Research Agent always does a live fetch of the
  announcement's own links first (Phase 2). This corpus just gives it
  broader context for the surrounding docs area.

### Phase 2 — RSS ingestion + Research Agent (the event pipeline)
- **RSS ingestion (event-driven)**: the RSS feed is published into Zerobus,
  landing as rows in a bronze Delta table (`rss_bronze`). New-row arrival on
  `rss_bronze` is the trigger for the Research Agent — either a
  file/table-arrival-triggered Job, or a step in the same Lakeflow
  Declarative Pipeline used for ingestion (Phase 2b below). This replaces a
  polling watcher entirely if Zerobus → bronze is viable end-to-end; falls
  back to a scheduled RSS poll + a `rss_seen` dedup table otherwise (see §6
  open question — flagged for investigation).
- **Research Agent** (one run per new `rss_bronze` row): per the
  architecture diagram above. Key design points:
  - Always does a LIVE fetch of the announcement body + every link it
    references — never relies solely on the Phase 1 corpus, since
    same-day announcements often link to pages not yet indexed.
  - Citations are first-class output, not an afterthought: every claim in
    the summary should be traceable to a specific fetched URL.
  - `ai_classify` needs a maintained taxonomy — the label set is "all
    currently installed/staged skill names + one-line descriptions," PLUS
    an always-present **"New Feature"** bucket for announcements that don't
    fit any existing skill well. This taxonomy must be kept in sync with
    the skill directory (regenerate it from `SKILL.md` frontmatter each
    run) and with any user-created buckets (see Phase 3's override flow).
  - Output is always a *proposal* written to a JSON findings file (never a
    direct write to a skill file or straight to Delta — see Phase 2b).

### Phase 2b — Ingestion pipeline (findings → queryable + searchable)
- A Lakeflow Declarative Pipeline (streaming, Autoloader) watches the JSON
  findings volume the Research Agent writes to and loads it: bronze (raw
  JSON) → silver (`research_log` Delta table, one row per skill
  classification per RSS entry) → embeds the proposed-diff/summary text and
  upserts the Vector Search index.
- This keeps the Research Agent task itself simple and idempotent (just
  write a file); all the "land it in tables and make it searchable" logic
  lives in one reusable streaming pipeline instead of being duplicated
  across every place that produces findings.

### Phase 3 — Review App (the human gate + editor)
- Databricks App (AppKit + React) reading `research_log` + the skill
  directory.
- Review queue UI: one card per pending research entry, with per-skill
  accept/reject, full diff view, and a markdown editor for hand-editing the
  proposed text before accepting.
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
  the label the human actually confirmed, not the model's first guess. A
  new bucket the user creates becomes a permanent taxonomy entry (and,
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
  CLI's own version string.
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
| RSS watcher + Research Agent | `databricks-jobs`, `databricks-ai-functions` (`ai_classify`, `ai_query`) |
| Review App | `databricks-apps`, `databricks-apps-python`, `databricks-app-design` |
| Knowledge base / MCP server | `databricks-vector-search`, `databricks-model-serving` (if fronting via an endpoint) |
| Bundling everything | `databricks-dabs` (ship Job + App as one bundle) |

## 4. Suggested build order

0. Phase 0 (workspace setup / repo configuration) is a prerequisite for
   this project's *own* development too, not just for future installers —
   even developing against this repo should exercise the same
   configuration path (pointed at this repo) rather than hardcoding it, so
   Phase 0's plumbing is proven from day one instead of retrofitted later.
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
4. **Upstream fix path** — **partially resolved as a side effect of #3**:
   the source-of-truth repo is confirmed public: `github.com/databricks/
   databricks-agent-skills`. So the mechanism for an upstream PR is no
   longer unknown — what's still undecided is whether this project should
   actually open PRs there once proposals are proven out. **Still marked as
   a nice-to-have, not a blocker per repo owner** — revisit after the core
   loop (Phases 1–5) works, but the path is no longer blocked on
   discovering where to send it.
5. ~~New-skill detection~~ — **Resolved**: `ai_classify`'s taxonomy always
   includes an explicit **"New Feature"** bucket for announcements that
   don't fit an existing skill well, and the App lets a user override any
   classification (including typing in a brand-new bucket name). Any
   override or new-bucket creation re-triggers skill-update generation for
   that entry. Reflected in Phase 2 and Phase 3 above.
6. **RSS feed identity**: confirm the exact feed URL (e.g. the release
   notes RSS/Atom endpoint) and whether it's one feed or several
   (per-product feeds) — affects whether ingestion is one Zerobus
   producer/pipeline or a fan-out per feed.
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
