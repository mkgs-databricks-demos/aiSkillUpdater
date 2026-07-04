# Review Before Copying — Changes in v0.2.0

This document lists all changes made to SKILL.md relative to the source at
`~/.databricks/aitools/skills/databricks-unstructured-pdf-generation/`.

## Files changed

### SKILL.md

**1. Version bump**
- `version: "0.1.0"` → `version: "0.2.0"`

**2. ai_parse_document DBR version corrected**
- All references to `17.1+` for ai_parse_document updated to `17.3+ (Serverless env v3+ required for VARIANT output)`
- Affected location: new "Parsing PDFs with ai_parse_document" section added to document the requirement explicitly

**3. New section: ai_prep_search (Beta, DBR 18.2+)**
- Added immediately after the ai_parse_document section
- Documents `ai_prep_search()` as the follow-on step after `ai_parse_document()` for preparing chunks for embedding and Vector Search indexing
- Includes a SQL example, input/output description, and DBR version requirement (18.2+ Serverless)

**4. mlflow.evaluate() → mlflow.genai.evaluate()**
- All mentions of `mlflow.evaluate()` updated to `mlflow.genai.evaluate()`
- Added deprecation note: "`mlflow.evaluate()` is deprecated as of MLflow 3.x — use `mlflow.genai.evaluate()` for GenAI evaluation"
- Affected locations: Step 4 workflow description and new "Evaluation with mlflow.genai.evaluate()" section

## Files copied unchanged

- `scripts/pdf_generator.py` — copied byte-for-byte from source
- `agents/openai.yaml` — copied byte-for-byte from source

## Files NOT copied (not needed in output)

- `assets/databricks.png` — binary asset, not modified; copy separately if needed
- `assets/databricks.svg` — SVG asset, not modified; copy separately if needed
