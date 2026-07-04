# REVIEW BEFORE COPYING — databricks-app-design (UPDATE-25)

Target: `~/.databricks/aitools/skills/databricks-app-design/`
Version: 0.1.0 → **0.2.0**

## Changes applied
1. **AppKit docs URL → developers.databricks.com/docs/appkit/v0/** — this skill had no github.io
   URL (uses `npx @databricks/appkit docs`); added an **AppKit web docs** note in
   `references/appkit-cheatsheet.md` pointing to `https://developers.databricks.com/docs/appkit/v0/`
   and marking the old github.io location deprecated.
2. **Verify `useGenieChat` hook and `genie()` scope names against current AppKit v0 docs** — added
   verify notes in `references/appkit-cheatsheet.md` and `references/genie-ai-trust.md`.
3. **Version → 0.2.0**.

## Reviewer checks
- No source URL was being changed (the skill relies on the AppKit CLI); the new web-docs URL is
  additive. Confirm `developers.databricks.com/docs/appkit/v0/` is the correct public docs path.

## Files in this update
- SKILL.md (modified: version only)
- references/appkit-cheatsheet.md (modified)
- references/genie-ai-trust.md (modified)
- other references/* (unchanged copies)
