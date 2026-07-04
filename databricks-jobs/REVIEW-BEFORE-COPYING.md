# REVIEW BEFORE COPYING — databricks-jobs (UPDATE-20)

Target: `~/.databricks/aitools/skills/databricks-jobs/`
Version: 0.2.0 (already) → **0.2.0** (unchanged; content corrections applied)

## Changes applied
1. **CLI floor v0.288.0 → v1.0.0** — the CLAUDE.md/AGENTS.md scaffold snippet's
   "Install the Databricks CLI (>= v0.288.0)" line now reads "(>= v1.0.0)".
   (The frontmatter `compatibility` was already `>= v1.0.0`.)
2. **'Databricks Asset Bundles (DABs)' → 'Declarative Automation Bundles (DABs)'** — the intro
   sentence ("managed via ...") and the "### Asset Bundles (DABs) — recommended" heading now use
   "Declarative Automation Bundles (DABs)". The scaffold note already read "Declarative Automation
   Bundles (formerly Databricks Asset Bundles)" and was left intact.
3. **'Lakeflow Jobs' as current product name in description** — already present in the frontmatter
   description and title ("# Lakeflow Jobs Development"); verified, no change needed.

## Reviewer notes
- This skill was already at 0.2.0 with most branding/CLI updates applied; this pass only fixes the
  two remaining literals (install-floor line + the "Asset Bundles" heading/intro).
- No changes were required in references/*.md (they use "DABs YAML" section labels, which are fine).

## Files in this update
- SKILL.md (modified)
- references/* (unchanged copies)
