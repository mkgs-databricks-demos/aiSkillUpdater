# Document Processing Pipeline with AI Functions

End-to-end patterns for building batch document processing pipelines using AI Functions in a Lakeflow Declarative Pipeline. Covers function selection, `config.yml` centralization, error handling, `ai_prep_search` for RAG chunking, and guidance on near-real-time variants with DSPy or LangChain.

---

## Runtime Requirements

All AI Functions require **Serverless compute**:

| Function | Minimum DBR | Notes |
|---|---|---|
| All task-specific functions | DBR 18.2+ | Serverless env required |
| `ai_parse_document` | DBR 17.3+ | Serverless env v3+ required for VARIANT |
| `ai_prep_search` | DBR 18.2+ | Beta since April 2026 |
| `ai_forecast` | DBR 18.2+ | Serverless SQL warehouse only |

> **Classic clusters and Classic/Pro SQL warehouses do not support AI Functions.** Use `serverless: true` in your pipeline or job configuration.

---

## Function Selection for Document Pipelines

When processing documents with AI Functions, apply this order of preference for each stage:

| Stage | Preferred function | Use `ai_query` when... |
|---|---|---|
| Parse binary docs (PDF, DOCX, images) | `ai_parse_document` | Need image-level reasoning |
| Prep chunks for RAG / vector search | `ai_prep_search` | — (always use this over manual explode) |
| Extract fields from text (flat or nested) | `ai_extract` | Schema exceeds 128 fields or 7 nesting levels |
| Classify document type or status | `ai_classify` | More than 500 categories |
| Score item similarity / matching | `ai_similarity` | Need cross-document reasoning |
| Summarize long sections | `ai_summarize` | — |
| Extract deeply nested JSON | `ai_query` with `responseFormat` | Schema exceeds `ai_extract` limits (128 fields, 7 levels) |

---

## Centralized Configuration (`config.yml`)

**Always centralize model names, volume paths, and prompts in a `config.yml`.** Foundation model endpoint names are versioned and rotated — a hardcoded name breaks silently when a model is retired.

```yaml
# config.yml
models:
  default: "databricks-claude-sonnet-4-5"
  mini:    "databricks-meta-llama-3-1-8b-instruct"
  vision:  "databricks-llama-4-maverick"
  embed:   "databricks-qwen3-embedding-0-6b"

catalog:
  name:   "my_catalog"
  schema: "document_processing"

volumes:
  input: "/Volumes/my_catalog/document_processing/landing/"
  tmp:   "/Volumes/my_catalog/document_processing/tmp/"

output_tables:
  results: "my_catalog.document_processing.processed_docs"
  errors:  "my_catalog.document_processing.processing_errors"

prompts:
  extract_invoice: |
    Extract invoice fields and return ONLY valid JSON.
    Fields: invoice_number, vendor_name, vendor_tax_id (digits only),
    issue_date (dd/mm/yyyy), total_amount (numeric),
    line_items: [{item_code, description, quantity, unit_price, total}].
    Return null for missing fields.

  classify_doc: |
    Classify this document into exactly one category.
```

```python
# config_loader.py
import yaml

def load_config(path: str = "config.yml") -> dict:
    with open(path) as f:
        return yaml.safe_load(f)

CFG           = load_config()
ENDPOINT      = CFG["models"]["default"]
ENDPOINT_MINI = CFG["models"]["mini"]
VOLUME_INPUT  = CFG["volumes"]["input"]
PROMPT_INV    = CFG["prompts"]["extract_invoice"]
```

---

## Batch Pipeline — Lakeflow Declarative Pipeline

Each logical step in your document workflow maps to a `@dlt.table` stage. Data flows through Delta tables between stages.

```
[Landing Volume]  →  Stage 1: ai_parse_document
                  →  Stage 2: ai_classify (document type)
                  →  Stage 3: ai_extract (flat fields) + ai_query (nested JSON)
                  →  Stage 4: ai_similarity (item matching)
                  →  Stage 5: Final Delta output table
```

### `pipeline.py`

```python
import dlt
import yaml
from pyspark.sql.functions import expr, col, from_json

CFG      = yaml.safe_load(open("/Workspace/path/to/config.yml"))
ENDPOINT = CFG["models"]["default"]
VOL_IN   = CFG["volumes"]["input"]
PROMPT   = CFG["prompts"]["extract_invoice"]


# ── Stage 1: Parse binary documents ──────────────────────────────────────────
# Preferred: ai_parse_document — no model selection, no ai_query needed

@dlt.table(comment="Parsed document text from all file types in the landing volume")
def raw_parsed():
    return (
        spark.read.format("binaryFile").load(VOL_IN)
        .withColumn("parsed", expr("ai_parse_document(content, MAP('version', '2.0'))"))
        .withColumn("text_blocks", expr("""
            concat_ws('\n', transform(
                parsed:document:elements,
                e -> e:content::STRING
            ))
        """))
        .selectExpr(
            "path",
            "text_blocks",
            "parsed:error_status AS parse_error",
        )
        .filter("parse_error IS NULL")
    )


# ── Stage 2: Classify document type ──────────────────────────────────────────
# Preferred: ai_classify v2 — cheap, returns VARIANT, no endpoint selection

@dlt.table(comment="Document type classification")
def classified_docs():
    return (
        dlt.read("raw_parsed")
        .withColumn(
            "doc_type",
            expr("""
                ai_classify(
                    text_blocks,
                    '["invoice", "purchase_order", "receipt", "contract", "other"]',
                    MAP('version', '2.0')
                ):response[0]::STRING
            """)
        )
    )


# ── Stage 3a: Flat field extraction ──────────────────────────────────────────
# Preferred: ai_extract for flat fields (vendor, date, total)

@dlt.table(comment="Flat header fields extracted from documents")
def extracted_flat():
    return (
        dlt.read("classified_docs")
        .filter("doc_type = 'invoice'")
        .filter("text_blocks IS NOT NULL")
        .withColumn(
            "result",
            expr("""
                ai_extract(
                    text_blocks,
                    '{
                        "invoice_number": {"type": "string"},
                        "vendor_name":    {"type": "string"},
                        "issue_date":     {"type": "string", "description": "dd/mm/yyyy"},
                        "total_amount":   {"type": "number"},
                        "tax_id":         {"type": "string"}
                     }',
                    MAP('version', '2.0')
                )
            """)
        )
        .selectExpr(
            "path", "doc_type", "text_blocks",
            "result:response AS header",
            "result:error_message::STRING AS extract_error"
        )
    )


# ── Stage 3b: Nested JSON extraction (last resort: ai_query) ─────────────────
# Use ai_query only for deeply nested schemas that exceed ai_extract's 7-level limit

@dlt.table(comment="Nested line items extracted — ai_query used for array schema only")
def extracted_line_items():
    return (
        dlt.read("extracted_flat")
        .filter("extract_error IS NULL")
        .withColumn(
            "ai_response",
            expr(f"""
                ai_query(
                    '{ENDPOINT}',
                    concat('{PROMPT.strip()}', '\\n\\nDocument text:\\n', LEFT(text_blocks, 6000)),
                    responseFormat => '{{"type":"json_object"}}',
                    failOnError     => false
                )
            """)
        )
        .withColumn(
            "line_items",
            from_json(
                col("ai_response.response"),
                "STRUCT<line_items:ARRAY<STRUCT<item_code:STRING, description:STRING, "
                "quantity:DOUBLE, unit_price:DOUBLE, total:DOUBLE>>>"
            )
        )
        .select("path", "doc_type", "header", "line_items", col("ai_response.errorMessage").alias("extraction_error"))
    )


# ── Stage 4: Similarity matching ─────────────────────────────────────────────
# Preferred: ai_similarity for fuzzy matching between extracted fields

@dlt.table(comment="Vendor name similarity vs reference master data")
def vendor_matched():
    extracted = dlt.read("extracted_line_items")
    vendors = spark.table("my_catalog.document_processing.vendor_master").select("vendor_id", "vendor_name")

    return (
        extracted.crossJoin(vendors)
        .withColumn(
            "name_similarity",
            expr("ai_similarity(header:vendor_name::STRING, vendor_name)")
        )
        .filter("name_similarity > 0.80")
        .orderBy("name_similarity", ascending=False)
    )


# ── Stage 5: Final output + error sidecar ────────────────────────────────────

@dlt.table(
    comment="Final processed documents ready for downstream consumption",
    table_properties={"delta.enableChangeDataFeed": "true"},
)
def processed_docs():
    return (
        dlt.read("extracted_line_items")
        .filter("extraction_error IS NULL")
        .selectExpr(
            "path",
            "doc_type",
            "header:invoice_number::STRING AS invoice_number",
            "header:vendor_name::STRING AS vendor_name",
            "header:issue_date::STRING AS issue_date",
            "header:total_amount::DOUBLE AS total_amount",
            "line_items.line_items AS items",
        )
    )


@dlt.table(comment="Rows that failed at any extraction stage — review and reprocess")
def processing_errors():
    return (
        dlt.read("extracted_flat")
        .filter("extract_error IS NOT NULL")
        .select("path", "doc_type", col("extract_error").alias("error"))
        .unionByName(
            dlt.read("extracted_line_items")
            .filter("extraction_error IS NOT NULL")
            .select("path", "doc_type", col("extraction_error").alias("error"))
        )
    )
```

---

## RAG Pipeline — Parse → Chunk → Index → Query

When the goal is retrieval-augmented generation rather than field extraction, use `ai_prep_search` (DBR 18.2+) to automatically produce context-enriched chunks ready for embedding. This replaces the manual `variant_explode_outer` + `concat_ws` approach.

### Recommended Pipeline (ai_prep_search)

```
ai_parse_document  →  ai_prep_search  →  embed chunk_to_embed  →  AI Search index
```

**Step 1 — Parse and prep chunks (Streaming):**

```python
from pyspark.sql.functions import col, current_timestamp, expr

files_df = (
    spark.readStream.format("binaryFile")
    .option("pathGlobFilter", "*.{pdf,jpg,jpeg,png,docx,pptx}")
    .option("recursiveFileLookup", "true")
    .load("/Volumes/catalog/schema/volume/docs/")
)

parsed_df = (
    files_df
    .repartition(8, expr("crc32(path) % 8"))          # parallelise parse across workers
    .withColumn("parsed", expr("""
        ai_parse_document(content, map(
            'version', '2.0',
            'descriptionElementTypes', '*'
        ))
    """))
    .filter("parsed:error_status IS NULL")
    .withColumn("parsed_at", current_timestamp())
    .select("path", "parsed", "parsed_at")
)

(
    parsed_df.writeStream.format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/Volumes/catalog/schema/checkpoints/01_parse")
    .option("mergeSchema", "true")
    .trigger(availableNow=True)
    .toTable("catalog.schema.parsed_documents_raw")
)
```

**Step 2 — Expand into chunks with ai_prep_search:**

```python
from pyspark.sql.functions import col, expr

parsed_stream = spark.readStream.format("delta").table("catalog.schema.parsed_documents_raw")

chunks_df = (
    parsed_stream
    .withColumn("chunks_raw", expr("ai_prep_search(parsed)"))
    .withColumn("chunk", expr("explode(chunks_raw)"))
    .select(
        "path",
        "parsed_at",
        col("chunk:chunk_to_embed").alias("chunk_to_embed"),   # context-enriched — EMBED THIS
        col("chunk:chunk_index").alias("chunk_index"),
        col("chunk:page_number").alias("page_number"),
        col("chunk:element_type").alias("element_type"),
        col("chunk:content").alias("raw_content"),
    )
    .filter("chunk_to_embed IS NOT NULL")
)

(
    chunks_df.writeStream.format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/Volumes/catalog/schema/checkpoints/02_chunks")
    .option("mergeSchema", "true")
    .trigger(availableNow=True)
    .toTable("catalog.schema.document_chunks")
)
```

**Step 3 — Enable CDF for AI Search Delta Sync:**

```sql
ALTER TABLE catalog.schema.document_chunks
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

**Step 4 — Create AI Search index on `chunk_to_embed`:**

Use the **[databricks-vector-search](../databricks-vector-search/SKILL.md)** skill (now "Databricks AI Search") to create a Delta Sync index on `document_chunks` with `chunk_to_embed` as the source column and `databricks-qwen3-embedding-0-6b` as the embedding model.

### Legacy Manual Chunking (Pre-DBR 18.2 Only)

> **Use `ai_prep_search` instead if running DBR 18.2+.** The manual approach below is preserved for clusters that cannot be upgraded.

`ai_parse_document` returns a VARIANT. Use `variant_get` with an explicit `ARRAY<VARIANT>` cast before calling `explode`, since `explode()` does not accept raw VARIANT values.

```sql
CREATE OR REPLACE TABLE catalog.schema.parsed_chunks AS
WITH parsed AS (
  SELECT
    path,
    ai_parse_document(content, map('version', '2.0')) AS doc
  FROM read_files('/Volumes/catalog/schema/volume/docs/', format => 'binaryFile')
),
elements AS (
  SELECT
    path,
    explode(variant_get(doc, '$.document.elements', 'ARRAY<VARIANT>')) AS element
  FROM parsed
)
SELECT
  md5(concat(path, variant_get(element, '$.content', 'STRING'))) AS chunk_id,
  path AS source_path,
  variant_get(element, '$.content', 'STRING') AS content,
  variant_get(element, '$.type', 'STRING') AS element_type,
  current_timestamp() AS parsed_at
FROM elements
WHERE variant_get(element, '$.content', 'STRING') IS NOT NULL
  AND length(trim(variant_get(element, '$.content', 'STRING'))) > 10;
```

---

## Near-Real-Time Variant — DSPy + MLflow Agent

When the pipeline must respond in seconds (triggered by a user action, API call, or form submission), use DSPy with an MLflow ChatAgent instead of a DLT pipeline.

**When to use DSPy vs LangChain:**

| Scenario | Stack |
|---|---|
| Fixed pipeline steps, well-defined I/O, want prompt optimization | **DSPy** |
| Needs tool-calling, memory, or multi-agent coordination | **LangChain LCEL** + MLflow ChatAgent |
| Single LLM call, simple task | Direct AI Function or `ai_query` in a notebook |

### DSPy Signatures (replace LangChain agent system prompts)

```python
# pip install dspy-ai mlflow databricks-sdk
import dspy, yaml

CFG = yaml.safe_load(open("config.yml"))
lm = dspy.LM(
    model=f"databricks/{CFG['models']['default']}",
    api_base="https://<workspace-host>/serving-endpoints",
    api_key=dbutils.secrets.get("scope", "databricks-token"),
)
dspy.configure(lm=lm)


class ExtractInvoiceHeader(dspy.Signature):
    """Extract invoice header fields from document text."""
    document_text:  str = dspy.InputField(desc="Raw text from the document")
    invoice_number: str = dspy.OutputField(desc="Invoice number, or null")
    vendor_name:    str = dspy.OutputField(desc="Vendor/supplier name, or null")
    issue_date:     str = dspy.OutputField(desc="Date as dd/mm/yyyy, or null")
    total_amount:  float = dspy.OutputField(desc="Total amount as float, or null")


class ClassifyDocument(dspy.Signature):
    """Classify a document into one of the provided categories."""
    document_text: str = dspy.InputField()
    category:      str = dspy.OutputField(
        desc="One of: invoice, purchase_order, receipt, contract, other"
    )


class DocumentPipeline(dspy.Module):
    def __init__(self):
        self.classify = dspy.Predict(ClassifyDocument)
        self.extract  = dspy.Predict(ExtractInvoiceHeader)

    def forward(self, document_text: str):
        doc_type = self.classify(document_text=document_text).category
        if doc_type == "invoice":
            header = self.extract(document_text=document_text)
            return {"doc_type": doc_type, "header": header.__dict__}
        return {"doc_type": doc_type, "header": None}


pipeline = DocumentPipeline()
```

### Wrap and Register with MLflow

```python
import mlflow, json

class DSPyDocumentAgent(mlflow.pyfunc.PythonModel):
    def load_context(self, context):
        import dspy, yaml
        cfg = yaml.safe_load(open(context.artifacts["config"]))
        lm = dspy.LM(model=f"databricks/{cfg['models']['default']}")
        dspy.configure(lm=lm)
        self.pipeline = DocumentPipeline()

    def predict(self, context, model_input):
        text = model_input.iloc[0]["document_text"]
        return json.dumps(self.pipeline(document_text=text), ensure_ascii=False)

mlflow.set_registry_uri("databricks-uc")
with mlflow.start_run():
    mlflow.pyfunc.log_model(
        artifact_path="document_agent",
        python_model=DSPyDocumentAgent(),
        artifacts={"config": "config.yml"},
        registered_model_name="my_catalog.document_processing.document_agent",
    )
```

---

## Common Issues

| Issue | Solution |
|-------|----------|
| AI Functions not found | Requires Serverless compute (DBR 18.2+). Classic clusters and Classic/Pro warehouses are not supported. |
| `ai_parse_document` not found | Requires DBR 17.3+ on Serverless env v3+. Check pipeline `channel: PREVIEW` and `serverless: true`. |
| `ai_prep_search` not found | Requires DBR 18.2+ on Serverless. Verify pipeline runtime channel. Beta since April 2026. |
| `explode()` fails with VARIANT | `explode()` requires ARRAY, not VARIANT. Use `ai_prep_search` to get a proper ARRAY, or use `variant_get(doc, '$.document.elements', 'ARRAY<VARIANT>')` to cast before exploding (legacy path). |
| Short/noisy chunks (legacy path) | Filter with `length(trim(...)) > 10` — parsing produces tiny fragments (page numbers, headers) that pollute the index. `ai_prep_search` handles this automatically. |
| Re-parsing unchanged documents | Use Structured Streaming with checkpoints — see Step 1 above. |
| Region not supported | US/EU regions only, or enable cross-geography routing. |
| `ai_classify` returns wrong type | Ensure you pass `map('version', '2.0')` and extract with `:response[0]::STRING`, not as a raw string column. |

---

## Tips

1. **Parse first, enrich second** — always run `ai_parse_document` as the first stage. Feed its VARIANT output directly to `ai_prep_search` (for RAG) or task-specific functions (for extraction); never pass raw binary to `ai_query`.
2. **Use `ai_prep_search` for all RAG chunking** — it produces context-enriched `chunk_to_embed` fields with surrounding headers, related questions, and table summaries baked in. Manual element-level explode produces lower-quality embeddings.
3. **Flat or nested fields → `ai_extract`; deeply nested JSON exceeding 7 levels → `ai_query`** — pass `MAP('version', '2.0')` and access results through `:response`.
4. **`failOnError => false` is mandatory in batch** — write errors to a sidecar `_errors` table rather than crashing the pipeline.
5. **Truncate before sending to `ai_query`** — use `LEFT(text, 6000)` or chunk long documents to stay within context window limits.
6. **Prompts belong in `config.yml`** — never hardcode prompt strings or model names in pipeline code. A prompt or model change should be a config change, not a code change.
7. **DSPy for agents** — when migrating from LangChain agent-based tools, DSPy typed `Signature` classes give you structured I/O contracts, testability, and optional prompt compilation/optimization.
