# Review Before Copying — databricks-metric-views Skill Updates

This file documents all changes made from the source files in
`~/.databricks/aitools/skills/databricks-metric-views/` to the updated files
in `~/skill-updates/databricks-metric-views/`.

---

## Summary of Changes

### All Files: YAML keyword `dimensions:` → `fields:`

In every YAML code block across all three files, the keyword `dimensions:` was
replaced with `fields:`. Prose text that uses the word "dimensions" as English
(e.g., "dimensions are the columns that...") was NOT changed. The replacement
applies only where `dimensions:` appears as a YAML key at the start of a line
(possibly indented) inside a fenced code block.

Specific occurrences replaced:

**SKILL.md** — 5 code blocks containing `dimensions:`:
- Quick Start example (lines 51–72 in source)
- SQL Operations > Create Metric View example
- CLI Execution bash/JSON example
- Convert an Existing View example
- YAML Spec Quick Reference block

**references/yaml-reference.md** — 4 code blocks:
- The main `fields:` (formerly `dimensions:`) section example
- Materialization example (two `dimensions:` sub-keys inside `materialized_views`)
- Complete Example at the bottom (both main block and materialization sub-block)

**references/patterns.md** — 9 pattern blocks each containing `dimensions:`:
- Pattern 1 (Simple Metrics)
- Pattern 2 (Derived Dimensions with CASE)
- Pattern 3 (Ratio Measures)
- Pattern 4 (Filtered Measures)
- Pattern 5 (Star Schema)
- Pattern 6 (Snowflake Schema)
- Pattern 7 (Materialized Metric View) — also materialization sub-key
- Pattern 8 (TPC-H Demo)
- Pattern 9 window sub-patterns (Trailing Window, Running Total, Day-Over-Day Growth, YTD, Bank Balance)
- SQL Examples section (Create with joins, Alter to add measure)

---

## SKILL.md — Specific Changes

### 1. Version bump
- `version: "0.1.0"` → `version: "0.2.0"`

### 2. YAML keyword change
- All `dimensions:` in code blocks replaced with `fields:` (see above)

### 3. Backward-compat note added
Added after the Quick Start code block:
> **Backward compatibility:** `dimensions:` is accepted as a legacy alias for `fields:` but `fields:` is now the canonical keyword.

### 4. YAML Spec Quick Reference updated
- Changed `dimensions:` key and its comment to:
  ```yaml
  fields:                         # Required: at least one (dimensions: accepted as legacy alias)
  ```

### 5. Key Concepts table renamed
- Section heading changed from "Dimensions vs Measures" to "Fields (Dimensions) vs Measures"
- Table column header "Dimensions" → "Fields (Dimensions)"
- Added explanatory note below the table about `fields:` being canonical and `dimensions:` being a legacy alias

### 6. Materialization: Experimental → Preview
- In YAML Spec Quick Reference: comment changed from `# Optional (experimental)` to `# Optional (Preview)`
- Added new "Materialization (Preview)" sub-section under Key Concepts explaining the two types:
  - **aggregated**: pre-aggregates metric values for faster queries
  - **unaggregated**: materializes raw rows for flexible downstream aggregation

### 7. Common Issues table updated
- Materialization row: "currently experimental" → "currently in Preview"

### 8. New DBR Version Ladder section added
Inserted after Prerequisites:

| DBR Version | Features |
|-------------|----------|
| DBR 16.4+ | Base metric views support (YAML v0.1) |
| DBR 17.3+ | Agent metadata support, TEMPORARY metric views |
| DBR 18.0+ | BI compatibility mode |
| DBR 18.1+ | Cardinality controls, offset windows |
| DBR 18.2+ | `agg()` alias support |

### 9. Reference Files table description updated
- "dimensions, measures, joins" → "fields, measures, joins"

### 10. Convert an Existing View section
- Prose updated: "promote `GROUP BY` columns to `dimensions`" → "promote `GROUP BY` columns to `fields`"

---

## references/yaml-reference.md — Specific Changes

### 1. Top-Level Fields table
- `dimensions` row description updated: "Array of dimension definitions (at least one)." →
  "Array of field (dimension) definitions (at least one). `dimensions:` is accepted as a legacy alias."

### 2. Section heading renamed
- "## Dimensions" → "## Fields (Dimensions)"

### 3. Backward-compat note added
Added note at top of Fields section:
> **Note:** `fields:` is the canonical keyword. `dimensions:` is accepted as a legacy alias for backward compatibility.

### 4. Field Rules section renamed
- "### Dimension Rules" → "### Field Rules"
- Rule text updated: "Can reference source columns, SQL functions, CASE expressions, and other dimensions" →
  "...and other fields"

### 5. New Semantic Metadata section added
Added after Field Rules, before Measures:

```yaml
# Semantic metadata (optional per field)
fields:
  - name: revenue
    display_name: "Total Revenue"
    synonyms: ["income", "sales", "rev"]
    formatting:
      type: currency
      currency_code: USD
      decimal_places: 2
```

With a table describing `display_name`, `synonyms`, and `formatting` keys.

### 6. Materialization section: Experimental → Preview
- Section heading: "## Materialization (Experimental)" → "## Materialization (Preview)"
- Opening description: "Pre-compute aggregations..." (unchanged)
- Materialization Types table descriptions updated:
  - `unaggregated`: added "as raw rows for flexible downstream aggregation"
  - `aggregated`: changed to "Pre-aggregates metric values for specific field/measure combinations for faster queries"

### 7. Materialization YAML example
- `dimensions:` sub-keys inside `materialized_views` blocks changed to `fields:`

### 8. Complete Example
- Main `dimensions:` block → `fields:`
- `materialized_views` sub-key `dimensions:` → `fields:`

---

## references/patterns.md — Specific Changes

### 1. YAML keyword change in all patterns
All `dimensions:` YAML keys replaced with `fields:` in all code blocks (Patterns 1–9 and SQL Examples).

### 2. Materialization: Experimental → Preview
- Pattern 7 section description updated: "Pre-compute common aggregations for faster queries."
  Added "Materialization is in **Preview**." at the end of the intro sentence.
- The `materialized_views` sub-key `dimensions:` → `fields:` in Pattern 7.

### 3. Section text in Pattern 2
- Section heading: "Pattern 2: Derived Dimensions with CASE" → "Pattern 2: Derived Fields with CASE"
- Intro: "Transform raw values into business-friendly categories." (unchanged)

### 4. Window pattern fields updated
- All window measure patterns (Pattern 9 sub-patterns) use `fields:` instead of `dimensions:`

---

## Files NOT changed

- `agents/openai.yaml` — not modified (out of scope)
- `assets/databricks.png` — not modified (binary asset)
- `assets/databricks.svg` — not modified (asset)
