# REVIEW BEFORE COPYING — databricks-agent-bricks (UPDATE-26)

Target: `~/.databricks/aitools/skills/databricks-agent-bricks/`
Version: 0.1.0 → **0.2.0**

## Changes applied
1. **CLI version contradiction fixed** — removed all `0.299.x` references (SKILL.md:60 and
   references/2-supervisor-agents.md:3) and use **CLI ≥ v1.0.0** consistently. (Frontmatter
   `compatibility` was already `>= v1.0.0`.)
2. **Supervisor-agents Beta → "Beta (verify current status — may be GA)"** in both the SKILL.md
   Supervisor Agent section and references/2-supervisor-agents.md.
3. **Version → 0.2.0**.

## Reviewer checks
- `grep -rn "0.299" .` returns nothing.
- Confirm the actual current status (Beta vs GA) of `databricks supervisor-agents` and tighten
  once verified.

## Files in this update
- SKILL.md (modified)
- references/2-supervisor-agents.md (modified)
- references/1-knowledge-assistants.md (unchanged copy)
