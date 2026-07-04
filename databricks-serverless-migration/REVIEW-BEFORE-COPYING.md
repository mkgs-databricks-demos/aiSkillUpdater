# REVIEW BEFORE COPYING — databricks-serverless-migration (UPDATE-22)

Target: `~/.databricks/aitools/skills/databricks-serverless-migration/`
Version: 0.1.0 → **0.2.0**

## Changes applied
1. **Environment version table: added v5 as current latest.**
   - `references/configuration-guide.md` "Environment Version Mapping" table now includes rows for
     `"4"` (JAR jobs) and `"5"` (**current**), with the explicit ladder
     **v1-legacy → v2 → v3 → v4 → v5-current**.
   - `SKILL.md` config section references the same ladder.
2. **Removed 'Native support coming soon' for `df.cache()`/`df.persist()` → marked UNVERIFIED.**
   - `SKILL.md` Category A table row + the "Remove caching" AFTER comment.
   - `references/compatibility-checks.md` `df.cache()` row.
3. **Added documented limits: 7-day max runtime, 128 MB local-rows cap for
   `spark.createDataFrame`.**
   - `references/configuration-guide.md`: new "Documented serverless limits" table.
   - `SKILL.md`: "Documented serverless limits" note pointing to that table.
4. **Foundation model examples: `llama-3-3-70b` is no longer 'the current default'.**
   - `SKILL.md` D1 blocker row + the troubleshooting 404 row now note Llama 4, GPT-5.x, and Qwen3
     are also available, with "no single current default" and a "verify endpoint exists" caveat.
5. **Version → 0.2.0**.

## Reviewer checks — VERIFY BEFORE PUBLISHING
- The **v5 = current latest** claim and the exact Python version per env version are best-effort;
  confirm against current serverless environment-version docs.
- The **7-day** max runtime and **128 MB** `createDataFrame` local cap are the documented values in
  the task spec; re-confirm against the current serverless limitations page.
- The `compatibility` CLI floor was left at `>= v0.292.0` (task did not request a change for this
  skill). Adjust if you want it aligned with the v1.0.0 floor used elsewhere.
- `cache()`/`persist()` is now explicitly labeled UNVERIFIED rather than "coming soon".

## Files in this update
- SKILL.md (modified)
- references/configuration-guide.md (modified)
- references/compatibility-checks.md (modified)
- other references/* (unchanged copies)
