# UPDATE-01: databricks-ai-functions — Review Notes

**Staging location:** `~/skill-updates/databricks-ai-functions/`  
**Target location:** `~/.databricks/aitools/skills/databricks-ai-functions/`  
**Date prepared:** 2026-07-03

To install after review:
```bash
cp ~/skill-updates/databricks-ai-functions/SKILL.md \
   ~/.databricks/aitools/skills/databricks-ai-functions/SKILL.md

cp ~/skill-updates/databricks-ai-functions/references/*.md \
   ~/.databricks/aitools/skills/databricks-ai-functions/references/
```

---

## Files Changed

| File | Change type | Summary |
|------|-------------|---------|
| `SKILL.md` | Updated | Prerequisites, quick-start syntax, ai_prep_search added, common issues corrected |
| `references/1-task-functions.md` | Updated + appended | DBR floor corrected, ai_classify v2 syntax hardened, ai_prep_search full section added |
| `references/2-ai-query.md` | Updated | Model roster refreshed to July 2026 names |
| `references/3-ai-forecast.md` | Updated | "Pro or Serverless" → "Serverless only" |
| `references/4-document-processing-pipeline.md` | Updated | Runtime requirements table added, ai_prep_search RAG pipeline replaces manual explode, old batch cluster recommendation removed |

---

## Critical Changes (Will Break Generated Code If Left as-is)

### 1. DBR Version Floor — All Files

| Was | Now | Impact |
|-----|-----|--------|
| "DBR 15.1+" | **DBR 18.2+** | Code sent to Classic cluster silently produces NULL |
| "DBR 15.4 ML LTS recommended for batch" | **REMOVED** | Classic cluster cannot run any AI Function at all |
| "DBR 17.1+ for ai_parse_document" | **DBR 17.3+ (Serverless env v3+)** | Wrong version floor |
| "Pro or Serverless warehouse" for ai_forecast | **Serverless only** | Pro warehouse is Classic infrastructure |

### 2. `ai_classify` v2 Syntax — SKILL.md + references/1-task-functions.md

Old (v1 — broken on v2-only clusters):
```sql
ai_classify(text, ARRAY('urgent', 'not urgent', 'spam'))  -- returns STRING
```

New (v2 — required):
```sql
ai_classify(text, '["urgent", "not urgent", "spam"]', map('version', '2.0')):response[0]::STRING
```

All examples updated. The ARRAY() form returns a bare STRING and does not support label descriptions, multilabel, or 500-label cap.

### 3. `ai_query` Model Names — references/2-ai-query.md

| Was | Now |
|-----|-----|
| `databricks-claude-sonnet-4` | `databricks-claude-sonnet-4-5` |
| `databricks-gte-large-en` | `databricks-qwen3-embedding-0-6b` |

Other models unchanged. Always load from config.yml in production.

---

## New Content Added

### `ai_prep_search` — Full New Section in references/1-task-functions.md

- Function signature, requirements (DBR 18.2+, Serverless, Beta April 2026)
- Chunk output shape: `chunk_to_embed`, `chunk_index`, `page_number`, `element_type`, `content`
- Python streaming pipeline: parse → ai_prep_search → explode → write chunks Delta table
- SQL equivalent for DLT tables
- Position in the RAG pipeline diagram

### RAG Pipeline Replacement in references/4-document-processing-pipeline.md

- New "Recommended Pipeline (ai_prep_search)" section replaces "Step 1" manual explode
- Two-stage streaming pipeline: parse → chunk with independent checkpoints
- Legacy manual chunking preserved under "Pre-DBR 18.2 Only" heading
- Runtime requirements table at top of file

### SKILL.md

- `ai_prep_search` added to Overview table and function selection rule table
- Pattern 3 updated: parse → ai_prep_search → chunk_to_embed → AI Search
- Prerequisites block fully rewritten
- Common issues table: 4 new rows, 3 corrected rows
- Reference files list updated to mention ai_prep_search
- Description frontmatter updated to include ai_prep_search and note RAG pipeline update
- Version bumped: 0.1.0 → 0.2.0

---

## What Was NOT Changed

- `references/3-ai-forecast.md` — only the "Pro or Serverless" → "Serverless only" wording; all patterns and parameters are correct
- `agents/openai.yaml` — not a Markdown update target; left untouched
- `assets/` — images untouched
