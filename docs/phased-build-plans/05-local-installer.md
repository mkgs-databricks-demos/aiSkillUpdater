# Build Plan 05 — Phase 5: Local Installer (get an approved skill onto the coding agent's machine)

**Source of truth:** `docs/PROJECT-PLAN.md` §2 Phase 5 (the Local Installer
design), §1a (bring-your-own-repo — the configured skills repo, `apply-skill.sh`
living in it, starter-pack seeding), §5 (version-tag frontmatter every written
file must carry for later drift detection), §1d (three-bundle split — the App
UI surface lives in `-app`). This plan restates what's needed to build Phase 5;
it does not re-litigate design decisions already locked there — if something
here seems to conflict with `PROJECT-PLAN.md`, that file wins and this plan is
stale.

**Cross-reviewed:** _pending_ — a first draft. An independent structural pass
(`pi`) and an independent Databricks/mechanics fact-check pass (`codex`) will
review this draft; every BLOCKING finding from both will be folded into a
revised version, with a changelog at the bottom.

**What this phase is, in one sentence:** the mechanism that takes a skill
change the human already approved in the Review App (Build Plan 04) and gets it
onto the local machine where their coding agent (Claude Code, Cursor, Codex
CLI, OpenCode, GitHub Copilot, Antigravity) actually reads skills from —
primarily via a small, reviewed-once `apply-skill.sh` the user runs after
`git pull`, with a downloadable one-off script as the fallback for users who
haven't cloned the repo.

**The one architectural fact that shapes everything here (design rationale, not
re-litigated):** a Databricks App backend runs in the workspace, **not on the
user's laptop** — it physically cannot write to `~/.claude/skills/` on the
reviewer's machine. So "apply locally" is never a server-side file write. Every
delivery mechanism in this plan is something that *runs on the user's own
machine* (a script they execute, or a browser API acting with their local
consent). The App's job is only to **produce and hand over** that script/content
and the instructions, plus the git PR path Build Plan 04 already owns. See
`PROJECT-PLAN.md` §2 Phase 5.

> **Decisions made in this plan that `PROJECT-PLAN.md` does not pin down.**
> Phase 5 specifies the *what* (git-based `apply-skill.sh` primary,
> downloadable one-off script fallback, per-agent paths, plugin-vs-raw
> detection, §5 provenance frontmatter, a rejected daemon, an optional
> File System Access button) but leaves several *hows* open. This plan makes
> explicit, flagged choices for: **how `apply-skill.sh` gets seeded into a
> from-scratch configured repo** (§5 Part A, §11 #1); the script's **input /
> idempotency / backup contract** (§5 Part A, §11 #2); the **plugin-vs-raw
> detection heuristic per agent** — the single biggest correctness risk here,
> currently *unverified* (§5 Part A step 4, §7, §11 #3); **Antigravity's
> skill-directory path, which `PROJECT-PLAN.md` does not actually document**
> even though Antigravity is in the target set (§4, §11 #4); and the
> **signed-URL mechanism** for the download fallback (§5 Part B, §11 #5). Each
> discretionary choice is consolidated in §11 for the repo owner and
> cross-review to confirm or override.

---

## 1. Objective

Make an approved skill change land on the coding agent's machine, safely and
with provenance. When this plan is done:

1. A reviewed-once **`apply-skill.sh`** lives in the configured skills repo
   (§1a). After a reviewer accepts a change (Build Plan 04 opens a PR → human
   merges → user `git pull`s), running this script fans the skill out to
   whichever supported agents are installed on that machine, in each agent's
   own directory/format.
2. The script **detects plugin-vs-raw mode per agent** before writing, so it
   never blindly drops raw files into a directory an agent's own plugin
   manager will later clobber.
3. Every file the script writes carries the **§5 version-tag frontmatter**, so
   a later CLI-drift check (Build Plan 06) can always identify provenance.
4. The Review App (Build Plan 04) exposes the **"apply to this machine"
   affordance** (the hook Build Plan 04 §5 Part E step 16 left as a seam) —
   showing the `git pull && ./apply-skill.sh` instruction for repo-cloned
   users, and a **downloadable one-off script** (short-TTL signed URL, shown
   before execution, never silent `curl | bash`) for users without the repo.
5. Optionally (Chromium only, convenience add-on), a **"grant folder access"**
   button uses the browser File System Access API to write the approved files
   with the user's explicit local consent, for users who won't touch a script
   or git.

**Out of scope (owned elsewhere or rejected):** the accept/PR flow itself
(Build Plan 04 — this plan consumes its hook and its "always PR" output); the
drift check that reads the provenance frontmatter (Build Plan 06); **Pi as a
target** (`PROJECT-PLAN.md` §2 Phase 5 — no CLI-side convention exists, not a
priority now); a **local companion daemon** (explicitly rejected in
`PROJECT-PLAN.md` §2 Phase 5 — disproportionate build/sign/distribute cost and
doesn't remove the code-execution concern, just relocates it).

## 2. Prerequisites

- **Build Plan 04 built and deployed.** This plan hard-depends on Build Plan 04
  §5 Part E step 16's accept-time hook (the "apply to this machine" affordance
  seam) and on Build Plan 04's "accept always opens a PR to the configured
  repo" behavior (git-based delivery *extends* that path; it does not replace
  it).
- **Build Plan 00 built and deployed.** The configured skills repo (§1a) and
  the starter-pack registry are Build Plan 00's; this plan's script-seeding
  question (§11 #1) is entangled with Build Plan 00's onboarding
  starting-point flow (scratch vs. starter pack).
- **The §5 versioning frontmatter scheme** is stable (defined in
  `PROJECT-PLAN.md` §5; Build Plan 04 §5 Part E step 15 already writes it into
  the PR'd files). The script re-asserts the same scheme on the files it
  writes locally, so provenance is consistent between the repo and the local
  copy.
- **A machine with one or more supported agents installed**, for realistic
  end-to-end testing (the detection logic can only be validated against real
  agent installs / plugin state).

## 3. Bundle placement (per `PROJECT-PLAN.md` §1d)

| Resource | Bundle | Notes |
|---|---|---|
| The "apply to this machine" App UI + download-fallback endpoint (§5 Part B) | `-app` | Part of the Databricks App (§1d places Phase 5 in `-app`). Extends Build Plan 04's App; not a new deployable. |
| `apply-skill.sh` itself | **not in any project bundle** | It lives in the *user's configured skills repo* (§1a), committed there (from the starter-pack seed, or added at first setup for a from-scratch repo — §11 #1). It is not shipped by this project's `-infra`/`-ai-tools`/`-app` bundles. This is the key placement subtlety: the primary deliverable is authored here but *deployed into someone else's repo*, not into a DAB. |
| Signed-URL storage for the download fallback (§5 Part B) | *open — §11 #5* | Depends on the signing mechanism chosen (e.g. a UC Volume with a short-TTL signed URL vs. an App-served ephemeral endpoint). |

**No new Lakebase tables.** This plan adds no `-infra`/`-ai-tools` resources
and no new OLTP state; it is a local-execution script plus an App affordance
over Build Plan 04's existing accept flow.

## 4. Delivery model & artifacts (what gets built, and where each runs)

Three delivery mechanisms, in priority order, all executing **on the user's
machine**:

1. **Primary — `apply-skill.sh` in the configured repo.** After merge + `git
   pull`, the user runs it locally. Security profile (`PROJECT-PLAN.md` §2
   Phase 5): the payload was already reviewed as a PR diff, and the only local
   execution is a small, stable, auditable, reviewed-once script — not a
   freshly generated blob per action.
2. **Fallback — downloadable one-off script** for users who haven't cloned the
   repo. Served from a **short-TTL signed URL**, scoped to file-writes only (no
   remote `eval`), and **shown to the user before execution** (never silent
   `curl | bash`).
3. **Optional convenience (Chromium only) — File System Access API button.**
   Writes the approved files with the user's explicit in-browser folder-grant
   consent. A convenience add-on, not the primary path.

**Per-agent raw-skill target paths** (confirmed from CLI source per
`PROJECT-PLAN.md` §2 Phase 5, except where noted):

| Agent | Path |
|---|---|
| Claude Code | `~/.claude/skills` (global) / `<cwd>/.claude/skills` (project) |
| Cursor | `~/.cursor/skills` |
| Codex CLI | `~/.codex/skills` |
| GitHub Copilot | `~/.copilot/skills` |
| OpenCode | `${XDG_CONFIG_HOME:-~/.config}/opencode/skills` (or `$APPDATA/opencode/skills` on Windows) |
| Antigravity | **NOT documented in `PROJECT-PLAN.md`** despite being in the target set — must be resolved from CLI source before the script can target it (§11 #4) |

**Every written file carries the §5 provenance frontmatter** (`metadata.version`,
`base_cli_version`, `base_skills_repo_ref`, `updated_at`, `last_research_log_id`,
`availability`) so Build Plan 06's drift check can identify what this installer
placed and from which base.

## 5. Task breakdown

### Part A — `apply-skill.sh` (authored here, committed into the configured repo)
1. **Agent detection**: detect which of the supported agents (Claude Code,
   Cursor, Codex CLI, OpenCode, GitHub Copilot, Antigravity) are present on the
   machine, by probing for each one's directory/config (§4 path table). Skip
   absent agents silently; never create an agent's tree just because the skill
   exists.
2. **Input / scope contract** (§11 #2): decide and document what the script
   operates on — a single named skill, a list, or a full sync of the repo's
   skills. Recommended: default to syncing all skills present in the repo (the
   repo is the source of truth post-merge), with an optional argument to scope
   to one skill. Make it **idempotent** (re-running with no repo change is a
   no-op).
3. **Back up before overwrite** (§11 #2): before overwriting an agent's
   existing skill file, preserve the prior copy (e.g. a timestamped `.bak` or a
   one-level backup dir) so a bad apply is recoverable — this is overwriting
   files the CLI or the user may have placed.
4. **Plugin-vs-raw detection per agent — the biggest correctness risk (§7,
   §11 #3).** The CLI's current default is **plugin-first** for Claude
   Code/Codex/Copilot (only Cursor/OpenCode/Antigravity get raw files by
   default), so blindly writing raw files into `~/.claude/skills/` can be
   silently overwritten/conflict on the next `aitools install`/plugin update.
   Before writing, detect whether each agent is plugin-managed (e.g. inspect
   the CLI's own `.state.json` / plugin records) and either (a) write raw only
   when the agent is in raw mode, or (b) warn + require an explicit
   `--force-raw` when it's plugin-managed. **This heuristic is unverified** —
   see §7 / §11 #3; it must be validated against real installs at build time.
5. **Write with provenance**: write each skill to the detected agents' paths,
   stamping the §5 frontmatter (§4). For Claude Code, default to the global
   `~/.claude/skills` unless a project-scope flag is given; handle the
   OpenCode Windows `$APPDATA` path.
6. **Report**: print a clear per-agent summary (applied / skipped-absent /
   skipped-plugin-managed / backed-up), so the user knows exactly what changed.

### Part B — App affordance + download fallback (`-app`, extends Build Plan 04)
7. Implement Build Plan 04 §5 Part E step 16's **"apply to this machine"
   affordance**: for a repo-cloned user, show the exact `git pull &&
   ./apply-skill.sh` (or scoped) command; do not attempt any server-side local
   write (impossible — §What-this-phase-is).
8. **Download fallback**: for users without the repo, generate the same script
   content as a one-off download from a **short-TTL signed URL** (§11 #5),
   file-writes only (no remote `eval`), and render it for the user to read
   **before** they run it — never a silent `curl | bash`.
9. Keep both affordances **gated behind an accepted item** (the PR/merge is the
   real gate); the installer never applies something that wasn't accepted in
   Build Plan 04.

### Part C — Optional File System Access API button (`-app`, Chromium-only, convenience)
10. Behind a clearly-optional, Chromium-only affordance, use the browser
    **File System Access API** to let the user grant folder access and write
    the approved files directly — with the same §5 frontmatter and the same
    plugin-vs-raw caution surfaced as a warning (the browser can't run the
    detection heuristic, so this path should warn the user to confirm the agent
    isn't plugin-managed). Explicitly optional; do not block the plan on it.

### Seeding `apply-skill.sh` into the configured repo (cross-cutting, §11 #1)
11. Decide and implement how the script arrives in a user's repo:
    - **Starter-pack repos** (e.g. "Matt's version"): the script is already
      present in the pack, so it comes with the seed at onboarding.
    - **From-scratch repos**: the script must be added at first setup — either
      the App commits it as part of onboarding (Build Plan 00), or it rides in
      as part of the first accepted PR (Build Plan 04). Pick one and record it
      (§11 #1); it is entangled with Build Plan 00's starting-point flow.

## 6. Cross-plan contracts (shared config — do not re-decide here)

- **Build Plan 04 accept hook** (its §5 Part E step 16): this plan implements
  the affordance Build Plan 04 left as a seam. Build Plan 04 owns accept→PR;
  this plan owns "now get the merged result onto the machine."
- **Configured skills repo + starter packs** (§1a / Build Plan 00 §4): the
  script lives in that repo; the starter-pack seed is how it arrives for
  pack-based installs. `installations` / `starter_packs` are Build Plan 00's.
- **§5 provenance frontmatter**: the script writes the *same* scheme Build Plan
  04 §5 Part E step 15 writes into PR'd files, so local copies and repo copies
  carry identical provenance for Build Plan 06's drift check to read.
- **Build Plan 06 (Phase 4) reads the provenance** the installer writes — this
  plan's only obligation to Build Plan 06 is to stamp every written file
  correctly (§4, §5 Part A step 5).

## 7. Platform constraints to build against (to be cross-review-confirmed)

- **A Databricks App backend cannot write to the user's local filesystem** —
  every apply mechanism must run on the user's machine (script or browser API).
  No server-side local write exists; the App only produces script/content +
  instructions.
- **Plugin-first default for Claude Code/Codex/Copilot** (`PROJECT-PLAN.md` §2
  Phase 5): raw-file writes to those agents can be clobbered by the next
  plugin/`aitools install` update. The plugin-vs-raw detection heuristic (§5
  Part A step 4) is the mitigation — **and is currently unverified**; confirm
  the actual detection signal (the CLI's `.state.json` / plugin-record
  location and shape) against a real install before relying on it. Treat as
  the top build-time verification item.
- **Antigravity path undocumented** (§4, §11 #4): in the target set but its
  skill directory isn't recorded anywhere in `PROJECT-PLAN.md`; resolve from
  CLI source before the script targets it, or the script must degrade to
  "skip Antigravity with a clear message" until it is known.
- **`curl | bash` safety** — the download fallback must be shown to the user
  before execution and scoped to file-writes only; no remote `eval`, no silent
  piping.
- **File System Access API is Chromium-only** — the Part C button must
  feature-detect and hide/disable on non-Chromium browsers rather than error.

## 8. Explicit fan-out map

**Within this plan:**
- **Part A (`apply-skill.sh`)** is the core artifact and is largely
  independent of Parts B/C — it can be authored and tested standalone against a
  real machine with agents installed. This is where the real risk lives
  (plugin-vs-raw, Antigravity path).
- **Part B (App affordance + download)** depends on Build Plan 04's App shell
  but not on Part A's internals — it can be built in parallel with Part A,
  wiring up the instruction/download UI, then pointed at the finished script.
- **Part C (File System Access button)** is optional and fully independent —
  fan out only if desired; never block the plan on it.
- **Script-seeding (step 11)** is a small cross-cutting decision that touches
  Build Plan 00's onboarding; resolve it before Part B's "repo-cloned" and
  "from-scratch" branches can both be exercised.

**Across plans:**
- **Blocked on Build Plan 04** (accept hook + PR flow) and **Build Plan 00**
  (configured repo, starter-pack seed, script-seeding decision). Hard
  sequential on Build Plan 04.
- **Feeds Build Plan 06** only via the provenance frontmatter it stamps.
- **Note**: `apply-skill.sh` is real executable code (bash). When this plan is
  *built*, that script is an `implement` task for a coding sub-agent → its own
  PR → cross-review → human merge, exactly like any other code deliverable.

## 9. Validation checklist

- [ ] On a machine with several supported agents installed, `apply-skill.sh`
      detects exactly those present and skips absent ones with a clear message.
- [ ] Each written skill lands in the correct per-agent path (§4), including
      Claude Code global-vs-project and OpenCode's `$APPDATA` case on Windows.
- [ ] Every written file carries correct §5 provenance frontmatter (same scheme
      as the repo copy).
- [ ] Re-running with no repo change is a no-op (idempotent); an existing file
      is backed up before it is overwritten.
- [ ] Plugin-vs-raw detection: on a **plugin-managed** Claude Code/Codex/Copilot
      install, the script does **not** blindly write raw files — it either skips
      with a warning or requires an explicit `--force-raw`. (Validate against a
      real plugin-managed install — §7 / §11 #3.)
- [ ] Antigravity is either targeted at a **verified** path or skipped with a
      clear "path unknown" message — never written to a guessed path (§11 #4).
- [ ] The App shows the correct instruction for a repo-cloned user, and a
      readable, pre-execution download for a non-cloned user from a short-TTL
      signed URL (no silent `curl | bash`).
- [ ] Every apply is gated behind an accepted/merged item — nothing unaccepted
      is ever installable.
- [ ] `apply-skill.sh` is present in a from-scratch repo via whatever seeding
      mechanism §11 #1 resolves to.
- [ ] (If built) the Chromium File System Access button feature-detects and is
      hidden/disabled on non-Chromium browsers.

## 10. Acceptance criteria (Definition of Done)

- `apply-skill.sh` exists in the configured repo, detects installed agents,
  writes each accepted skill to the right per-agent path with §5 provenance,
  is idempotent, backs up before overwrite, and reports per-agent results.
- Plugin-vs-raw is handled (not blind raw writes) with the detection heuristic
  verified against a real install, or — if verification is still pending — the
  script defaults to the safe behavior (warn + require `--force-raw`) rather
  than risking a clobber.
- Antigravity is handled per §11 #4 (verified path or explicit skip).
- The Review App exposes the "apply to this machine" affordance (repo-cloned
  instruction + download fallback), gated behind accepted items, with the
  download shown before execution.
- The daemon is not built; Pi is not targeted (both per `PROJECT-PLAN.md` §2
  Phase 5).
- Every §11 open item is either resolved and recorded here, or explicitly
  carried forward with its downstream owner.

## 11. Open items carried forward (decisions to confirm in cross-review)

1. **How `apply-skill.sh` is seeded into a from-scratch configured repo (§5
   step 11).** Starter-pack repos already carry it; from-scratch repos need it
   added at first setup (App commits it during Build Plan 00 onboarding vs. it
   rides in on the first accepted PR from Build Plan 04). Pick one — entangled
   with Build Plan 00's starting-point flow.
2. **Script input / idempotency / backup contract (§5 Part A steps 2–3).**
   Single-skill vs. list vs. full-sync default; the exact backup scheme
   (timestamped `.bak` vs. a backup dir); confirm idempotency semantics.
3. **Plugin-vs-raw detection heuristic per agent (§5 Part A step 4, §7).** The
   biggest correctness risk. The exact detection signal (the CLI's
   `.state.json` / plugin-record path and shape for Claude Code/Codex/Copilot)
   is **unverified** and must be validated against a real plugin-managed install
   at build time. Until verified, default to the safe behavior (warn + require
   `--force-raw`).
4. **Antigravity's skill-directory path (§4).** In the target agent set but
   **not documented in `PROJECT-PLAN.md`**; resolve from CLI source before the
   script targets it, or the script skips Antigravity with a clear message.
5. **Signed-URL mechanism for the download fallback (§3, §5 Part B).** How the
   App serves the one-off script over a short-TTL signed URL — UC Volume signed
   URL vs. an App-served ephemeral endpoint — and where that content is stored.
6. **File System Access API button (Part C).** Whether to build the optional
   Chromium-only convenience path at all in this phase, or defer it — it is
   explicitly optional and non-blocking.

## 12. Changelog

- **Initial draft** (pre-cross-review): drafted from `PROJECT-PLAN.md` §2
  Phase 5, §1a, §5, §1d, and the accept-hook contract in Build Plan 04 §5 Part
  E step 16. Led with the architectural constraint that a Databricks App
  backend cannot reach the user's laptop filesystem, so all apply mechanisms
  run on the user's machine. Made explicit flagged decisions for the six open
  items in §11 — most notably surfacing two genuine gaps: **Antigravity's
  skill path is undocumented in `PROJECT-PLAN.md`** despite being in scope
  (§4 / §11 #4), and the **plugin-vs-raw detection heuristic is unverified**
  and is the top build-time correctness risk (§5 Part A step 4 / §7 / §11 #3).
