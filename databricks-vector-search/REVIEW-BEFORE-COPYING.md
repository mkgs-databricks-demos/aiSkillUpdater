# Review Before Copying: databricks-vector-search → databricks-ai-search

## Files Changed

1. `SKILL.md`
2. `references/index-types.md`
3. `references/end-to-end-rag.md`
4. `references/search-modes.md`
5. `references/troubleshooting-and-operations.md`

---

## What Changed and Why

### SKILL.md

| Change | Why |
|--------|-----|
| `name:` → `databricks-ai-search`, added `aliases: [databricks-vector-search]` | Product rename |
| Version `0.1.0` → `0.2.0` | This update |
| All mentions of "Databricks Vector Search", "Mosaic AI Vector Search", "Vector Search" as product name → "Databricks AI Search" | Product rename |
| `pip install databricks-vectorsearch` → `pip install databricks-ai-search` | Package rename |
| `from databricks.vector_search.client import VectorSearchClient` → `from databricks.ai_search.client import AISearchClient` | Package rename |
| Added Full-Text Search Index (Beta) row to index types matrix | New 4th index type |
| Added Full-Text Search Index common pattern with code example | New 4th index type |
| Embedding model table: added `databricks-qwen3-embedding-0-6b` as **Production default**, relegated GTE/BGE to "legacy (still available)" | Embedding model update |
| Updated `embedding_model_endpoint_name` in Quick Start example to `databricks-qwen3-embedding-0-6b` | Embedding model update |
| Updated Storage-Optimized endpoint note: also required for Full-Text Search | New index type requirement |
| Added Full-Text Search endpoint error to Common Issues | New index type |
| Added `ImportError: databricks.vector_search` → install `databricks-ai-search` to Common Issues | Package rename |

### references/index-types.md

| Change | Why |
|--------|-----|
| Title → "Databricks AI Search Index Types" | Product rename |
| Added Full-Text Search Index (Beta) column to comparison matrix | New 4th index type |
| Added complete "Full-Text Search Index (Beta)" section with requirements, create code, query code, when-to-use, limitations | New 4th index type |
| Updated decision tree to include Full-Text Search branch | New 4th index type |
| Updated endpoint selection table to include Full-Text Search row | New 4th index type |
| Updated self-managed embedding example to use `databricks-qwen3-embedding-0-6b` | Embedding model update |
| Updated Direct Access attach-model example to use `databricks-qwen3-embedding-0-6b` | Embedding model update |
| All `VectorSearchClient` → `AISearchClient`, `databricks.vector_search` → `databricks.ai_search` | Package rename |
| Embedding dimension in self-managed examples: 768 → 1024 (matches qwen3-embedding-0-6b) | Accuracy |

### references/end-to-end-rag.md

| Change | Why |
|--------|-----|
| Added Step 0 section: "Context-Enriched Chunking with ai_prep_search (Recommended)" | New `ai_prep_search` feature (DBR 18.2+, Beta Apr 2026) |
| `ai_prep_search` note: DBR 18.2+, Beta since Apr 2026, `chunk_to_embed` is the field to index | Requirement from update spec |
| Updated embedding model in Step 3 to `databricks-qwen3-embedding-0-6b` | Embedding model update |
| Added `ai_prep_search` not available issue to Common Issues table | New feature |
| Title references updated to "Databricks AI Search" | Product rename |

### references/search-modes.md

| Change | Why |
|--------|-----|
| Title → "Databricks AI Search Modes" | Product rename |
| Added Full-Text Search mode section with when-to-use, example code using `AISearchClient` | New mode documentation |
| Updated decision table: FULL_TEXT row now specifies "requires Storage-Optimized endpoint + Full-Text index" | Accuracy for new index type |
| Updated FULL_TEXT notes in "Using with Pre-Computed Embeddings" section | Accuracy |
| Updated parameter reference table: FULL_TEXT note clarifies it uses Full-Text index | Accuracy |
| All `VectorSearchClient` → `AISearchClient`, `databricks-vectorsearch` → `databricks-ai-search` | Package rename |

### references/troubleshooting-and-operations.md

| Change | Why |
|--------|-----|
| Title → "Databricks AI Search Troubleshooting & Operations" | Product rename |
| Storage-Optimized recommendation note: "also required for Full-Text Search indexes" | New index type |
| Updated embedding model in migration example to `databricks-qwen3-embedding-0-6b` | Embedding model update |
| Added Full-Text Search endpoint error row to Expanded Troubleshooting table | New index type |
| Added `ImportError: databricks.vector_search` row | Package rename |
| All `databricks-vectorsearch` → `databricks-ai-search` | Package rename |

---

## Copy Commands

```bash
# Copy all updated files to the installed skill directory
SKILL_DIR="$HOME/.databricks/aitools/skills/databricks-vector-search"
UPDATE_DIR="$HOME/skill-updates/databricks-vector-search"

cp "$UPDATE_DIR/SKILL.md"                                            "$SKILL_DIR/SKILL.md"
cp "$UPDATE_DIR/references/index-types.md"                           "$SKILL_DIR/references/index-types.md"
cp "$UPDATE_DIR/references/end-to-end-rag.md"                        "$SKILL_DIR/references/end-to-end-rag.md"
cp "$UPDATE_DIR/references/search-modes.md"                          "$SKILL_DIR/references/search-modes.md"
cp "$UPDATE_DIR/references/troubleshooting-and-operations.md"        "$SKILL_DIR/references/troubleshooting-and-operations.md"

echo "Done. Verify with: head -5 $SKILL_DIR/SKILL.md"
```
