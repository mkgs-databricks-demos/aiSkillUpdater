# Review Before Copying: databricks-ml-training

## Files Changed

1. `SKILL.md`
2. `references/custom-pyfunc.md`
3. `references/genai-agents.md`

---

## What Changed and Why

### SKILL.md

| Change | Why |
|--------|-----|
| Version `0.1.0` → `0.2.0` | This update |
| Added Python 3.10 minimum note after opening description | SDK v0.103.0+ (Apr 2026) dropped Python 3.8/3.9 |
| **Added "MLflow 3 LoggedModel Paradigm" section** | New MLflow 3 API paradigm |
| Added: models can be logged without starting a run via `mlflow.initialize_logged_model(name=..., params=...)` | New API |
| Added: `artifact_path` deprecated in MLflow 3; use `name=` instead | Deprecation |
| Added: new canonical URI `models:/<model_id>`; old `runs:/` still works | New URI format |
| Added: key new APIs table: `mlflow.set_active_model()`, `mlflow.initialize_logged_model()`, `mlflow.finalize_logged_model()`, `mlflow.search_logged_models()`, `mlflow.create_external_model()` | MLflow 3 APIs |
| Added: backward compat note for `mlflow.xgboost.log_model(registered_model_name=...)` | Backward compat |
| **Added "mlflow.evaluate() Deprecation" section** | Deprecated in MLflow 3.0.0 |
| Deprecation warning: use `mlflow.models.evaluate()` for traditional ML or `mlflow.genai.evaluate()` for GenAI; will be removed in MLflow 3.7.0 | Requirement |
| Updated training notebook: `mlflow==3.1.0` → `mlflow==3.14.0` in job dependencies | MLflow version update |
| Updated training notebook: `artifact_path="model"` → `name="model"` in log_model call | MLflow 3 API |
| Added `client: "4"` note: env v5 is now available as latest; added v5 description | Serverless env v5 |
| Updated `servers env v5` note in jobs submit json comment | Env v5 |
| Added `mlflow.evaluate()` deprecation warning to Gotchas table | Deprecation |
| Added `Python 3.8/3.9 import errors` to Gotchas table | Python 3.10 minimum |
| Added `artifact_path=` deprecation to Gotchas table | MLflow 3 |

### references/custom-pyfunc.md

| Change | Why |
|--------|-----|
| `mlflow==3.1.0` → `mlflow==3.14.0` in `pip_requirements` | MLflow version update |
| `artifact_path="model"` → `name="model"` in `log_model` call | MLflow 3 deprecation |
| Added comment `# MLflow 3: use name= instead of artifact_path=` | Clarity |

### references/genai-agents.md

| Change | Why |
|--------|-----|
| `mlflow==3.1.0` → `mlflow==3.14.0` in `pip_requirements` | MLflow version update |
| `artifact_path="agent"` → `name="agent"` in `log_model` call | MLflow 3 deprecation |
| Added comment `# MLflow 3: use name= instead of artifact_path=` | Clarity |
| Updated LLM_ENDPOINT constant: `databricks-claude-sonnet-4-6` → `databricks-claude-sonnet-5` | Model roster update (current endpoint) |
| Updated OpenAI query example: `openai.OpenAI(...)` + old base_url pattern → `from databricks.openai import DatabricksOpenAI` | Updated client per model-serving deprecation |
| Updated Vector Search reference in resources table: `DatabricksVectorSearchIndex` description → "Vector Search / AI Search index" | Product rename alignment |

---

## Copy Commands

```bash
# Copy all updated files to the installed skill directory
SKILL_DIR="$HOME/.databricks/aitools/skills/databricks-ml-training"
UPDATE_DIR="$HOME/skill-updates/databricks-ml-training"

cp "$UPDATE_DIR/SKILL.md"                          "$SKILL_DIR/SKILL.md"
cp "$UPDATE_DIR/references/custom-pyfunc.md"       "$SKILL_DIR/references/custom-pyfunc.md"
cp "$UPDATE_DIR/references/genai-agents.md"        "$SKILL_DIR/references/genai-agents.md"

echo "Done. Verify with: head -5 $SKILL_DIR/SKILL.md"
```
