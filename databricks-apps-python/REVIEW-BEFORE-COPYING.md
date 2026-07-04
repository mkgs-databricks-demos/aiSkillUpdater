# REVIEW BEFORE COPYING — databricks-apps-python (UPDATE-23)

Target: `~/.databricks/aitools/skills/databricks-apps-python/`
Version: 0.1.0 → **0.2.0**

## Changes applied
1. **Pre-installed framework versions.**
   - `SKILL.md` Quick Reference "Pre-installed" row: replaced pinned versions with "verify current
     in Databricks Apps runtime docs"; **Gradio bumped to 5.x** (major version bump from 4.44.0);
     added a note to check current pre-installed versions before pinning.
   - `references/3-frameworks.md`: Dash "Pre-installed version" now "verify current (was 2.18.1)";
     Gradio "Pre-installed version" now "5.x (major bump from 4.44.0) — verify current".
2. **Removed 'preview' from Lakebase — GA.**
   - `references/5-lakebase.md`: "Lakebase is in Public Preview" → "Lakebase is GA".
3. **Added Jobs plugin to plugin reference table** (`SKILL.md` AppKit plugins table): "Jobs — trigger
   and monitor Lakeflow Jobs from the app".
4. **AppKit docs URL: github.io → developers.databricks.com/docs/appkit/v0/** (all three doc links +
   the Documentation section AppKit link).
5. **CLI floor v0.294.0/v0.295.0 → v1.0.0** (AppKit Requirements line; frontmatter was already v1.0.0).
6. **Version → 0.2.0**.

## Reviewer checks — VERIFY BEFORE PUBLISHING
- Streamlit/Flask/FastAPI pinned versions in `references/3-frameworks.md` were left as-is (task only
  called out Dash + Gradio); consider aligning them to the "verify current" wording too.
- Gradio "5.x" is per the task spec — confirm the exact pre-installed version in current Apps
  runtime docs.
- The "User auth — Public Preview" wording was left unchanged here (that GA-verify note is scoped to
  UPDATE-24 databricks-apps). Confirm if you want it aligned here too.
- Confirm the AppKit "Jobs" plugin name/capabilities against current AppKit v0 plugin docs.

## Files in this update
- SKILL.md (modified)
- references/3-frameworks.md (modified)
- references/5-lakebase.md (modified)
- other references/* and examples/* (unchanged copies)
