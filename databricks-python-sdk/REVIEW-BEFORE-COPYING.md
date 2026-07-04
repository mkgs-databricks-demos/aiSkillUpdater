# REVIEW BEFORE COPYING — databricks-python-sdk (UPDATE-21)

Target: `~/.databricks/aitools/skills/databricks-python-sdk/`
Version: 0.1.0 → **0.2.0**

## Changes applied
1. **Deprecation notice for `get_open_ai_client()`** (deprecated Feb 2026) → use
   `from databricks.openai import DatabricksOpenAI` (`pip install databricks-openai`).
   - `SKILL.md` serving section: added a comment block + a blockquote deprecation note next to the
     `w.serving_endpoints.get_open_ai_client()` example.
   - `examples/5-serving-and-vector-search.py`: added the same deprecation guidance inline at the
     `get_open_ai_client()` call site.
2. **Python 3.10 minimum** — Environment Setup now states the SDK requires Python 3.10+ (SDK
   v0.103.0+ dropped 3.8/3.9).
3. **Version → 0.2.0**.

## Reviewer checks
- The legacy `get_open_ai_client()` calls were kept (annotated as deprecated) rather than deleted,
  so existing readers still see the migration path. Remove the legacy line if you prefer a clean
  cut-over.
- Confirm the `DatabricksOpenAI().client` accessor shape matches the current `databricks-openai`
  package API before publishing (the import path `from databricks.openai import DatabricksOpenAI`
  is per the task spec).

## Files in this update
- SKILL.md (modified)
- examples/5-serving-and-vector-search.py (modified)
- examples/* and references/* (unchanged copies)
