# Review Before Copying: databricks-model-serving

## Files Changed

1. `SKILL.md`
2. `references/off-platform-streaming.md`

---

## What Changed and Why

### SKILL.md

| Change | Why |
|--------|-----|
| Version `0.1.0` → `0.2.0` | This update |
| Removed `workspace_client.serving_endpoints.get_open_ai_client()` examples | Deprecated Feb 2026 |
| Added `pip install databricks-openai` + `from databricks.openai import DatabricksOpenAI` + `client = DatabricksOpenAI()` section with streaming example | Replacement for deprecated client |
| Added deprecation notice: "DEPRECATED Feb 2026 — use DatabricksOpenAI() instead" under old client section | Deprecation notice requirement |
| Removed `workspace_client.serving_endpoints.get_langchain_chat_open_ai_client()` | Deprecated Feb 2026 |
| Added `pip install databricks-langchain` + `from databricks_langchain import ChatDatabricks` + `llm = ChatDatabricks(endpoint=...)` section | Replacement for deprecated LangChain client |
| Added deprecation notice for old LangChain client | Deprecation notice requirement |
| Updated Foundation Model endpoint roster table: added all current endpoints (July 2026) | Model roster update |
| Listed RETIRED endpoints: claude-3.7-sonnet, gemini-3-pro, llama-3.1-405b, llama-3, DBRX | Model roster update |
| Added "IMPORTANT: Never hardcode endpoint names" warning in FM API section | Best practice |
| Updated embedding default in "Defaults when user doesn't specify": `databricks-gte-large-en` → `databricks-qwen3-embedding-0-6b` | Model roster update |
| Added "AI Functions Optimized Models Mode" section | New feature |
| Added "Prompt Caching" section | New capability note |
| Added `ImportError: get_open_ai_client` and `ImportError: get_langchain_chat_open_ai_client` to Troubleshooting table | Deprecation guidance |

### references/off-platform-streaming.md

| Change | Why |
|--------|-----|
| Updated `DATABRICKS_ENDPOINT` env var comment: example changed from `databricks-meta-llama-3-3-70b-instruct` to `databricks-claude-sonnet-5` + added "Never hardcode endpoint names" note | Model roster update |
| Updated embeddings section: `databricks-gte-large-en` description → "Legacy, still available"; added `databricks-qwen3-embedding-0-6b` as "production default" | Model roster update |
| Updated OpenAI-compatible query example in embeddings: endpoint reference updated | Model roster update |
| Added "Endpoint name not found" row to Troubleshooting table | Guidance for model roster changes |

---

## Copy Commands

```bash
# Copy all updated files to the installed skill directory
SKILL_DIR="$HOME/.databricks/aitools/skills/databricks-model-serving"
UPDATE_DIR="$HOME/skill-updates/databricks-model-serving"

cp "$UPDATE_DIR/SKILL.md"                                      "$SKILL_DIR/SKILL.md"
cp "$UPDATE_DIR/references/off-platform-streaming.md"          "$SKILL_DIR/references/off-platform-streaming.md"

echo "Done. Verify with: head -5 $SKILL_DIR/SKILL.md"
```
