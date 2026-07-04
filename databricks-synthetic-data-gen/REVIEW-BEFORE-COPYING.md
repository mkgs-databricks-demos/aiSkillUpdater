# Review Before Copying — databricks-synthetic-data-gen v0.2.0

This document summarizes all changes made to the skill files relative to the v0.1.0 source.

## Files Changed

### SKILL.md

**Version bump:**
- `version: "0.1.0"` → `version: "0.2.0"`

**Faker version in Setup section:**
- `uv pip install "databricks-connect>=16.4,<17.4" faker numpy pandas holidays`
  changed to:
  `uv pip install "databricks-connect>=16.4,<17.4" faker==37.0.0 numpy pandas holidays`

**New section added — LLM-Based Synthetic Generation (DBR 18.2+ Serverless):**
- Added after the existing "Use Databricks Connect Spark + Faker Pattern" section.
- Covers `ai_gen()` for individual fields (SQL example).
- Covers `ai_query()` for full structured records (Python example).

**New section added — When to Use Faker vs ai_gen():**
- Decision matrix table added after the LLM section.
- Covers 8 use cases with tool recommendation and rationale.

### references/1-data-patterns.md

**No existing content changed.**

**New section added at end of file — LLM-Based Synthetic Generation Patterns (DBR 18.2+ Serverless):**
- Pattern 1: Mixed Faker + ai_gen() dataset using `spark.range()` with `F.expr("ai_gen(...)")`.
- Pattern 2: `ai_query()` for fully coherent records (RAG test data, chatbot evaluation datasets).

### references/2-troubleshooting.md

**Faker version references updated (2 occurrences):**
- In the ModuleNotFoundError solutions table: `uv pip install faker` → `uv pip install faker==37.0.0`
- In the inline comment under the table: updated install instruction to `faker==37.0.0`
- In the "Either base environment or version must be provided" job spec example: `"faker"` → `"faker==37.0.0"`

**New section added at end of file — ai_gen() / ai_query() Troubleshooting:**
- ai_gen() not found: requires DBR 18.2+ Serverless.
- Rate limiting guidance for large datasets (>100K rows).
- Slow generation: expectation-setting on Faker vs ai_gen() throughput.
- JSON parsing errors: use `try_parse_json()` + filter nulls pattern.
- Cost guidance: ~100 tokens per generated field value.

### scripts/generate_synthetic_data.py

**Copied unchanged from source.**

## Files NOT Changed

- `agents/openai.yaml` — not included in output (no changes requested).
- `assets/databricks.png` — binary asset, not included (no changes requested).
- `assets/databricks.svg` — asset, not included (no changes requested).
