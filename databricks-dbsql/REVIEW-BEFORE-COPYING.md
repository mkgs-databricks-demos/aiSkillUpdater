# REVIEW BEFORE COPYING — databricks-dbsql (UPDATE-18)

Target: `~/.databricks/aitools/skills/databricks-dbsql/`
Version: 0.1.0 → **0.2.0**

## Changes applied

1. **`BEGIN ATOMIC...END` clarified as SQL Scripting compound-statement / stored-procedure
   construct, NOT general transaction syntax.**
   - `SKILL.md` Quick Reference row relabeled "Transactions (SQL Scripting)" + a dedicated note.
   - `references/sql-scripting.md` "SQL Scripting Atomic Blocks": added a **Scope** callout
     directing client-driven transactions to the Python connector `autocommit = False` pattern.

2. **DBR 17.3 LTS and Preview channel (19.x features) added to version tables.**
   - `SKILL.md`: new "Databricks Runtime / DBSQL Channel Reference" table.
   - `references/sql-scripting.md`: two rows added to "Runtime Version Reference".

3. **Materialized view refreshes backed by a serverless pipeline, not a warehouse class.**
   - `SKILL.md` Key Guidelines: new bullet.
   - `references/materialized-views-pipes.md`: overview bullet strengthened to say the refresh
     is backed by a serverless (Lakeflow) pipeline, not a SQL warehouse class.

4. **Version → 0.2.0** in `SKILL.md` frontmatter.

## Reviewer checks
- The source already stated MVs use "serverless pipelines"; this update makes the "not a
  warehouse class" distinction explicit — confirm wording matches current product behavior.
- "Preview channel (19.x features)" is described as opt-in early access; verify the exact channel
  name/versioning against current DBSQL release-channel docs before publishing.

## Files in this update
- SKILL.md (modified)
- references/sql-scripting.md (modified)
- references/materialized-views-pipes.md (modified)
- references/{ai-functions,best-practices,geospatial-collations}.md (unchanged copies)
