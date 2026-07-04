---
name: databricks-unstructured-pdf-generation
description: "Build RAG / unstructured-document evaluation datasets and demo documents (e.g. for Knowledge Assistant) on Databricks: generate synthetic PDFs locally, upload to Unity Catalog volumes, and pair each document with test questions for retrieval evaluation."
compatibility: Requires databricks CLI (>= v1.0.0)
metadata:
  version: "0.2.0"
parent: databricks-core
---

# Unstructured-Document for Demos and Eval Datasets on Databricks

Workflow for producing **synthetic PDF documents + paired test questions** as a Unity Catalog-resident dataset for Demos and RAG / unstructured-document retrieval evaluation on Databricks. The PDF-generation step uses standard local HTML → PDF tooling; the Databricks-specific value is the workflow shape — UC volume layout, paired question files, and integration with downstream Databricks retrieval / `ai_extract` / `ai_parse_document` evaluation.

## Workflow

1. Write HTML files to `./raw_data/html/` (write multiple files in parallel for speed) — domain-shaped to match the documents your retrieval pipeline will see in production.
2. Convert HTML → PDF using `<SKILL_ROOT>/scripts/pdf_generator.py` (parallel conversion, wraps `plutoprint`).
3. Upload PDFs to a Unity Catalog volume via `databricks fs cp` — same volume shape your production pipeline will read from.
4. Generate `doc_questions.json` pairing each document with retrieval-eval questions; this becomes the gold dataset for `mlflow.genai.evaluate()` or comparable retrieval-quality scorers.

> If you only need ad-hoc PDFs (no Databricks workflow), any HTML → PDF tool (`weasyprint`, `wkhtmltopdf`, `playwright pdf`, `plutoprint`) works directly — this skill exists for the synthetic-dataset-on-UC end-to-end shape, not as a general PDF generator.

> **Path convention:** `<SKILL_ROOT>` below = the directory containing this SKILL.md. Resolve to the absolute install path (e.g. `~/.claude/skills/databricks-unstructured-pdf-generation`). `./raw_data/...` paths are relative to your own project cwd.

## Dependencies

```bash
uv pip install plutoprint
```

## Step 1: Write HTML Files

```bash
mkdir -p ./raw_data/html
```

Write HTML documents to `./raw_data/html/filename.html`. Use subdirectories to organize (structure is preserved).

## Step 2: Convert to PDF

```bash
# Convert entire folder (parallel, 4 workers)
python <SKILL_ROOT>/scripts/pdf_generator.py convert --input ./raw_data/html --output ./raw_data/pdf
```

Skips files where PDF exists and is newer than HTML. Use `--force` to reconvert all.

## Step 3: Upload to Volume

`databricks fs` requires the `dbfs:` scheme prefix even for UC Volume paths. `-r` copies the *contents* of the source directory into the target (the source directory name is not preserved), so files land directly under `raw_data/`.

```bash
databricks fs cp -r --overwrite ./raw_data/pdf dbfs:/Volumes/my_catalog/my_schema/raw_data
```

## Step 4: Generate Test Questions

Create `./raw_data/pdf/pdf_eval_questions.json` with questions for Knowledge Assistant evaluation or MAS:

```json
{
  "api_errors_guide.pdf": {
    "question": "What is the solution for error ERR-4521?",
    "expected_fact": "Call /api/v2/auth/refresh with refresh_token before the 3600s TTL expires"
  },
  "installation_manual.pdf": {
    "question": "What port does the service use by default?",
    "expected_fact": "Port 8443 for HTTPS, configurable via CONFIG_PORT environment variable"
  }
}
```

This JSON can be used to build KA test cases and validate retrieval accuracy.

## Document Content Guidelines

When generating documents for Knowledge Assistant testing or demos:

- **Multi-page documents**: Each PDF should be several pages with substantial content
- **Specific error codes and solutions**: Include product-specific error codes, causes, and resolution steps
- **Technical details**: API endpoints, configuration parameters, version numbers, specific commands
- **Simple CSS**: Keep styling minimal for fast HTML creation and reliable PDF conversion
- **Queryable facts**: Include details a KA must read the document to answer (not general knowledge)

**Good document types:**
- Product user manuals with troubleshooting sections
- API error reference guides (error codes, causes, solutions)
- Installation/configuration guides with specific steps
- Technical specifications with version-specific details

**Example content:** Instead of generic "Connection failed" errors, write:
- "Error ERR-4521: OAuth token expired. Cause: Token TTL exceeded 3600s default. Solution: Call `/api/v2/auth/refresh` with your refresh_token before expiration. See Section 4.2 for token lifecycle management."

## CLI Reference

```
python <SKILL_ROOT>/scripts/pdf_generator.py convert [OPTIONS]

  --input, -i     Input HTML file or folder (required)
  --output, -o    Output folder for PDFs (required)
  --force, -f     Force reconvert (ignore timestamps)
  --workers, -w   Parallel workers (default: 4)
```

## Folder Structure

Subfolder structure is preserved:

```
./raw_data/html/                    ./raw_data/pdf/
├── report.html             →       ├── report.pdf
├── quarterly/                      ├── quarterly/
│   └── q1.html             →       │   └── q1.pdf
└── legal/                          └── legal/
    └── terms.html          →           └── terms.pdf
```

## Parsing PDFs with ai_parse_document

After uploading PDFs to a UC volume, use `ai_parse_document()` to extract structured content for downstream RAG or evaluation pipelines.

`ai_parse_document` requires **DBR 17.3+ (Serverless env v3+ required for VARIANT output)**.

```sql
SELECT
  path,
  ai_parse_document(path) AS parsed_content
FROM (
  SELECT 'dbfs:/Volumes/my_catalog/my_schema/raw_data/' || filename AS path
  FROM my_table
)
```

- Input: file path (UC volume or DBFS)
- Output: VARIANT containing structured document content (text, tables, metadata)
- Requires DBR 17.3+ (Serverless env v3+ required for VARIANT output)

### ai_prep_search (Beta, DBR 18.2+)
Use `ai_prep_search()` after `ai_parse_document()` to prepare document chunks for embedding and search indexing.

```sql
SELECT 
  doc_id,
  ai_prep_search(parsed_content) AS chunk_to_embed
FROM parsed_documents;
```

- Input: parsed document content (output of ai_parse_document)
- Output: optimized text chunks ready for embedding
- Use before inserting into Vector Search indexes
- DBR 18.2+ (Serverless) required

## Evaluation with mlflow.genai.evaluate()

> **Note:** `mlflow.evaluate()` is deprecated as of MLflow 3.x — use `mlflow.genai.evaluate()` for GenAI evaluation.

Use `mlflow.genai.evaluate()` with the paired `pdf_eval_questions.json` to score retrieval quality:

```python
import mlflow

results = mlflow.genai.evaluate(
    data=eval_dataset,          # built from pdf_eval_questions.json
    predict_fn=retrieval_fn,    # your RAG retrieval function
    scorers=[mlflow.genai.scorers.RetrievalGroundedness()],
)
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "plutoprint not installed" | `uv pip install plutoprint` |
| PDF looks wrong | Check HTML/CSS syntax |
| "Volume does not exist" | `databricks volumes create CATALOG SCHEMA VOLUME_NAME MANAGED` (four separate positional args, not `catalog.schema.volume`) |
