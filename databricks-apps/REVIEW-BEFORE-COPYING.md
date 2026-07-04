# REVIEW BEFORE COPYING — databricks-apps (UPDATE-24)

Target: `~/.databricks/aitools/skills/databricks-apps/`
Version: 0.1.2 → **0.2.0**

## Changes applied
1. **Jobs plugin in plugin list/table** — already present: `references/appkit/overview.md` plugin
   table has a **Jobs** row, and `SKILL.md` lists the `jobs()` plugin + a Jobs Guide link
   (`references/appkit/jobs.md`). Verified; no change needed.
2. **CLI floor v0.294.0 → v1.0.0** — `SKILL.md` frontmatter `compatibility`.
3. **User auth 'Public Preview' → verify-GA note** — `references/platform-guide.md` troubleshooting
   row for `user token passthrough not enabled` now says "(Public Preview — **verify current GA
   status**, this may now be GA)".
4. **AppKit docs URL → developers.databricks.com/docs/appkit/v0/** — this skill had no github.io
   URL (it uses `npx @databricks/appkit docs`); added an **AppKit web docs** note pointing to
   `https://developers.databricks.com/docs/appkit/v0/` and marking the old github.io location
   deprecated.
5. **Version → 0.2.0**.

## Reviewer checks
- Confirm the Jobs plugin coverage is sufficient for your needs (it was already documented).
- Verify the current GA status of Apps user authorization and tighten the wording once confirmed.

## Files in this update
- SKILL.md (modified)
- references/platform-guide.md (modified)
- other references/* (unchanged copies, incl. references/appkit/jobs.md and overview.md)
