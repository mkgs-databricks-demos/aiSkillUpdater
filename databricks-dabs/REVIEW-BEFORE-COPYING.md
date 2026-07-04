# REVIEW BEFORE COPYING — databricks-dabs (UPDATE-17)

Target: `~/.databricks/aitools/skills/databricks-dabs/`
Version: 0.1.x (no version field) → **0.2.0**

## Changes applied

1. **Removed `bundle validate --strict`** — replaced with `bundle validate` + `bundle summary`
   throughout.
   - `SKILL.md` General Guidelines #1: now `bundle validate` then `bundle summary`.
   - `references/deploy-and-run.md` Validation section rewritten: `bundle validate` /
     `bundle validate -t prod` plus `bundle summary` / `bundle summary -t prod`, with a note
     that `--strict` is no longer used.
   - `references/deploy-and-run.md` "Diagnosing Errors" and "Common Issues" table entries updated
     to drop `--strict`.

2. **CLI floor 0.281.0 / 0.283.0 → v1.0.0+**
   - `SKILL.md`: added `compatibility: Requires databricks CLI (>= v1.0.0)` and a CLI floor note.
   - `references/bundle-structure.md`: dataset_catalog/dataset_schema note now "requires CLI v1.0.0+".
   - `references/deploy-and-run.md`: dashboard dataset_catalog note now "CLI v1.0.0+".

3. **`.lvdash.json` exclusion note added**
   - `SKILL.md`: new guideline #7 + a dedicated section showing `sync.exclude: - '*.lvdash.json'`.

4. **Version → 0.2.0** — added `metadata.version: "0.2.0"` (file previously had no version field).

## Reviewer checks
- Confirm no remaining `--strict` occurrences: `grep -rn "strict" .` should return nothing.
- Confirm the `sync.exclude` YAML matches your bundle's existing sync block style.
- The 0.283.0 floor mentioned in the task did not appear in the source files; only 0.281.0/0.239.0
  historical notes existed. Both historical references were reframed against the v1.0.0 floor.

## Files in this update
- SKILL.md (modified)
- references/deploy-and-run.md (modified)
- references/bundle-structure.md (modified)
- references/{alerts,resource-permissions,sdp-pipelines}.md (unchanged copies)
