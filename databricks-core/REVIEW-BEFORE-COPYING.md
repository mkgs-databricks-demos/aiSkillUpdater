# REVIEW BEFORE COPYING — databricks-core (UPDATE-28)

Target: `~/.databricks/aitools/skills/databricks-core/` (SKILL.md + 3 .md files)
Version: 0.1.0 → **0.2.0**

## Changes applied
1. **CLI floor → v1.0.0 (current v1.6.0) in ALL core files.**
   - `SKILL.md`: `compatibility` `>= v0.292.0` → `>= v1.0.0`; the "missing or outdated (< v0.292.0)"
     STOP rule now reads "< v1.0.0" and notes current is v1.6.0.
   - `databricks-cli-install.md`: "recent version" precondition now specifies ">= v1.0.0 (current
     v1.6.0)".
   - `data-exploration.md`: authenticated-CLI prerequisite now notes ">= v1.0.0; current v1.6.0".
   - `databricks-cli-auth.md`: notes reference CLI v1.x / v1.6.0 (see below).
2. **Auth: `databricks auth login` as the recommended command in CLI v1.x** (alongside
   `databricks configure`).
   - `databricks-cli-auth.md`: added a "Recommended command (CLI v1.x)" note under the OAuth
     section (prefer `databricks auth login`; `databricks configure` remains for token/PAT setup).
   - `SKILL.md`: added an Authentication note in the command reference section.
3. **Branding: 'Databricks Asset Bundles' → 'Declarative Automation Bundles'.**
   - `SKILL.md`: bundles comment + a "Bundle branding" note ("Declarative Automation Bundles,
     formerly Databricks Asset Bundles").
4. **.databrickscfg format updates in CLI v1.0.**
   - `databricks-cli-auth.md`: added a "`.databrickscfg` format note (CLI v1.0)" — let the CLI
     manage the file; re-run `databricks auth login` after upgrading from a pre-v1.0 CLI.
5. **Version → 0.2.0**.

## Reviewer checks — VERIFY BEFORE PUBLISHING
- The auth reference file already used `databricks auth login` throughout (OAuth); the added notes
  make it the *explicitly recommended* command. `databricks configure` was not previously mentioned
  — confirm you want it referenced as the token/PAT alternative.
- The exact `.databrickscfg` format changes in CLI v1.0 are described generically ("format has
  evolved"); confirm/replace with the specific field changes if you have them documented.
- "Current is v1.6.0" is stated in several places — update if the current CLI has moved on.

## Files in this update
- SKILL.md (modified)
- databricks-cli-auth.md (modified)
- databricks-cli-install.md (modified)
- data-exploration.md (modified)
