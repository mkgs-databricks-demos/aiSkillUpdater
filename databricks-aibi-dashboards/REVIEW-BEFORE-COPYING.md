# REVIEW BEFORE COPYING — databricks-aibi-dashboards (UPDATE-27)

Target: `~/.databricks/aitools/skills/databricks-aibi-dashboards/`
Version: 0.2.0 (already) → **0.2.0** (content corrections applied)

## Changes applied
1. **Verify note for `workspace-entity-tag-assignments`** — added a note next to the
   `create-tag-assignment` example (SKILL.md) directing the reader to confirm the command against
   `databricks workspace --help` (and `databricks workspace-entity-tag-assignments --help`) in
   CLI v1.6.0.
2. **`aitools` is plugin-first in CLI v1.6.0** — added a note in the Quick Reference area:
   the invocation may be `databricks aitools ...` (plugin) rather than
   `databricks experimental aitools ...`; verify with `databricks aitools --help`.
3. **Quick reference consistency** — the Quick Reference lakeview commands already match the
   Implementation section; the new plugin-first note reconciles the `experimental aitools` vs
   `aitools` naming so the quick reference and guidelines stay consistent across CLI versions.
4. **CLI floor v1.0.0** — already present in frontmatter `compatibility`; verified.
5. **Version → 0.2.0** — already at 0.2.0; verified.

## Reviewer checks — VERIFY BEFORE PUBLISHING
- Run `databricks aitools --help` and `databricks experimental aitools --help` on the target CLI
  (v1.6.0) and confirm which subcommand path is exposed; tighten the notes accordingly.
- Confirm `workspace-entity-tag-assignments create-tag-assignment` syntax on CLI v1.6.0.
- This skill was already at 0.2.0/v1.0.0; only advisory verify-notes were added (no destructive
  changes to widget specs).

## Files in this update
- SKILL.md (modified)
- references/* (unchanged copies)
