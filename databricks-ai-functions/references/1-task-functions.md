# Task-Specific AI Functions — Full Reference

These functions require no model endpoint selection. They call pre-configured Foundation Model APIs optimized for each task. All require **DBR 18.2+** on **Serverless compute only** — Classic SQL warehouses and Classic clusters are NOT supported. `ai_parse_document` requires DBR 17.3+ (Serverless env v3+ required for VARIANT output support).

---

## `ai_analyze_sentiment`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_analyze_sentiment

Returns one of: `positive`, `negative`, `neutral`, `mixed`, or `NULL`.

```sql
SELECT ai_analyze_sentiment(review_text) AS sentiment
FROM customer_reviews;
```

```python
from pyspark.sql.functions import expr
df = spark.table("customer_reviews")
df.withColumn("sentiment", expr("ai_analyze_sentiment(review_text)")).display()
```

---

## `ai_classify`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_classify

**Syntax:** `ai_classify(content, labels [, options])`
- `content`: VARIANT | STRING — raw text, or VARIANT from `ai_parse_document` / `ai_extract`
- `labels`: STRING — JSON labels definition:
  - Simple array: `'["urgent", "not_urgent", "spam"]'`
  - With descriptions: `'{"billing_error": "Payment, invoice, or refund issues", "product_defect": "Any malfunction or bug"}'` (descriptions up to 1000 chars each)
  - 2–500 labels, each 1–100 characters
- `options`: optional MAP\<STRING, STRING\>:
  - `version`: `'2.0'` — **always pass this**; v2 returns VARIANT, v1 (legacy) returns STRING
  - `instructions`: task context to improve accuracy (max 20,000 chars)
  - `multilabel`: `"true"` to return multiple matching labels (default `"false"`)

**Returns VARIANT** (v2, required). Shape: `{"response": ["label1"], "error_message": null}`.
Extract the first label with `:response[0]::STRING`. Returns `NULL` if content is `NULL`.

> ⚠️ **v1 legacy syntax uses `ARRAY(...)` and returns a bare STRING.** Do not use for new pipelines.
> Always pass `map('version', '2.0')` to get the VARIANT return and unlock label descriptions, multilabel, and 500-label support.

```sql
-- simple labels (v2)
SELECT ticket_text,
       ai_classify(
           ticket_text,
           '["urgent", "not urgent", "spam"]',
           map('version', '2.0')
       ):response[0]::STRING AS priority
FROM support_tickets;

-- labels with descriptions + instructions (best accuracy)
SELECT ticket_text,
       ai_classify(
           ticket_text,
           '{"billing_error": "Payment, invoice, or refund issues",
             "product_defect": "Any malfunction, bug, or breakage",
             "account_issue": "Login failures, password resets"}',
           map(
               'version',      '2.0',
               'instructions', 'Customer support tickets for a SaaS product'
           )
       ):response[0]::STRING AS category
FROM support_tickets;

-- multilabel (a ticket can match multiple categories)
SELECT ticket_text,
       ai_classify(
           ticket_text,
           '["fee", "withdrawal", "deposit", "transfer", "adjustment"]',
           map('version', '2.0', 'multilabel', 'true')
       ):response AS transaction_types    -- returns VARIANT array of all matching labels
FROM bank_transactions;
```

```python
from pyspark.sql.functions import expr
df = spark.table("support_tickets")
df.withColumn(
    "priority",
    expr("ai_classify(ticket_text, '[\"urgent\", \"not urgent\", \"spam\"]', map('version', '2.0')):response[0]::STRING")
).display()
```

**Tips:**
- Use label descriptions (JSON object form) for ambiguous categories — they significantly improve accuracy
- `multilabel: "true"` enables multi-label classification without running multiple calls
- Up to 500 labels supported (v2 only)
- Always extract with `:response[0]::STRING` — dot notation returns NULL on VARIANT

---

## `ai_extract`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_extract

**Syntax:** `ai_extract(content, schema [, options])`
- `content`: VARIANT | STRING — raw text, or VARIANT from `ai_parse_document`
- `schema`: STRING — JSON schema definition:
  - Simple (field names only): `'["invoice_id", "vendor_name", "total_amount"]'`
  - Advanced (with types and descriptions):
    ```json
    {
      "invoice_id": {"type": "string"},
      "total_amount": {"type": "number"},
      "currency": {"type": "enum", "labels": ["USD", "EUR", "GBP"]},
      "line_items": {"type": "array", "items": {"type": "object", "properties": {...}}}
    }
    ```
  - Supported types: `string`, `integer`, `number`, `boolean`, `enum`
  - Max 128 fields, 7 nesting levels, 500 enum values
- `options`: optional MAP\<STRING, STRING\>:
  - `version`: `'2.1'` — pin the current version for stability
  - `instructions`: task context to improve extraction quality (max 20,000 chars)

Returns VARIANT `{"response": {...}, "error_message": null}`. Returns `NULL` if content is `NULL`.

**Versions** — the current `ai_extract` (a JSON-string schema, e.g. `'["brand", "model"]'`, optionally pinned with `options => map('version', '2.1')`) is **v2.1, which returns a VARIANT, not a STRUCT** — so you access its fields with colon notation. The legacy `ai_extract(content, ARRAY('brand', 'model'))` array-of-labels form (v1) returns a STRUCT accessed with dot notation. Prefer the JSON-string → VARIANT (v2.1) form for new pipelines.

**Accessing the result** — `ai_extract` returns a `VARIANT` (the extracted fields live under `:response`), **not a `STRUCT`**. Navigate it with the colon (`:`) path operator and cast the leaf. **Dot notation (`result.response.brand`) returns `NULL` silently on a VARIANT column — always use colon (`:`).**

```sql
WITH extracted AS (
  SELECT product_name, ai_extract(product_name, '["brand", "model"]') AS result
  FROM products
)
SELECT
  result:response:brand::STRING AS brand,    -- colon path, then cast
  result:response:model::STRING AS model,
  result:error_message::STRING  AS extract_error
FROM extracted;
```

```sql
-- simple schema
SELECT ai_extract(
    'Invoice #12345 from Acme Corp for $1,250.00',
    '["invoice_id", "vendor_name", "total_amount"]'
) AS extracted;
-- {"response": {"invoice_id": "12345", "vendor_name": "Acme Corp", ...}, "error_message": null}

-- composable with ai_parse_document
WITH parsed AS (
  SELECT ai_parse_document(content, MAP('version', '2.0')) AS parsed
  FROM READ_FILES('/Volumes/finance/invoices/', format => 'binaryFile')
)
SELECT ai_extract(
    parsed,
    '["invoice_id", "vendor_name", "total_amount"]',
    MAP('instructions', 'These are vendor invoices.')
) AS invoice_data
FROM parsed;
```

```python
from pyspark.sql.functions import expr
df = spark.table("messages")
df = df.withColumn(
    "entities",
    expr("ai_extract(message, '[\"person\", \"location\", \"date\"]')")
)
df.display()
```

---

## `ai_fix_grammar`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_fix_grammar

**Syntax:** `ai_fix_grammar(content)` — Returns corrected STRING.

Optimized for English. Useful for cleaning user-generated content before downstream processing.

```sql
SELECT ai_fix_grammar(user_comment) AS corrected FROM user_feedback;
```

```python
from pyspark.sql.functions import expr
df = spark.table("user_feedback")
df.withColumn("corrected", expr("ai_fix_grammar(user_comment)")).display()
```

---

## `ai_gen`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_gen

**Syntax:** `ai_gen(prompt)` — Returns a generated STRING.

Use for free-form text generation where the output format doesn't need to be structured. For structured JSON output, use `ai_query` with `responseFormat`.

```sql
SELECT product_name,
       ai_gen(CONCAT('Write a one-sentence marketing tagline for: ', product_name)) AS tagline
FROM products;
```

```python
from pyspark.sql.functions import expr
df = spark.table("products")
df.withColumn(
    "tagline",
    expr("ai_gen(concat('Write a one-sentence marketing tagline for: ', product_name))")
).display()
```

---

## `ai_mask`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_mask

**Syntax:** `ai_mask(content, labels)`
- `content`: STRING — text with sensitive data
- `labels`: ARRAY\<STRING\> — entity types to redact

Returns text with identified entities replaced by `[MASKED]`.

Common label values: `'person'`, `'email'`, `'phone'`, `'address'`, `'ssn'`, `'credit_card'`

```sql
SELECT ai_mask(
    message_body,
    ARRAY('person', 'email', 'phone', 'address')
) AS message_safe
FROM customer_messages;
```

```python
from pyspark.sql.functions import expr
df = spark.table("customer_messages")
df.withColumn(
    "message_safe",
    expr("ai_mask(message_body, array('person', 'email', 'phone'))")
).write.format("delta").mode("append").saveAsTable("catalog.schema.messages_safe")
```

---

## `ai_similarity`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_similarity

**Syntax:** `ai_similarity(expr1, expr2)` — Returns a FLOAT between 0.0 and 1.0.

Use for fuzzy deduplication, search result ranking, or item matching across datasets.

```sql
-- Deduplicate company names (similarity > 0.85 = likely duplicate)
SELECT a.id, b.id, a.name, b.name,
       ai_similarity(a.name, b.name) AS score
FROM companies a
JOIN companies b ON a.id < b.id
WHERE ai_similarity(a.name, b.name) > 0.85
ORDER BY score DESC;
```

```python
from pyspark.sql.functions import expr
df = spark.table("product_search")
df.withColumn(
    "match_score",
    expr("ai_similarity(search_query, product_title)")
).orderBy("match_score", ascending=False).display()
```

---

## `ai_summarize`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_summarize

**Syntax:** `ai_summarize(content [, max_words])`
- `content`: STRING — text to summarize
- `max_words`: INTEGER (optional) — word limit; default 50; use `0` for uncapped

```sql
-- Default (50 words)
SELECT ai_summarize(article_body) AS summary FROM news_articles;

-- Custom word limit
SELECT ai_summarize(article_body, 20)  AS brief   FROM news_articles;
SELECT ai_summarize(article_body, 0)   AS full    FROM news_articles;
```

```python
from pyspark.sql.functions import expr
df = spark.table("news_articles")
df.withColumn("summary", expr("ai_summarize(article_body, 30)")).display()
```

---

## `ai_translate`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_translate

**Syntax:** `ai_translate(content, to_lang)`
- `content`: STRING — source text
- `to_lang`: STRING — target language code

**Supported languages:** `en`, `de`, `fr`, `it`, `pt`, `hi`, `es`, `th`

For unsupported languages, use `ai_query` with a multilingual model endpoint.

```sql
-- Single language
SELECT ai_translate(product_description, 'es') AS description_es FROM products;

-- Multi-language fanout
SELECT
    description,
    ai_translate(description, 'fr') AS description_fr,
    ai_translate(description, 'de') AS description_de
FROM products;
```

```python
from pyspark.sql.functions import expr
df = spark.table("products")
df.withColumn(
    "description_es",
    expr("ai_translate(product_description, 'es')")
).display()
```

---

## `ai_parse_document`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_parse_document

**Requires:** DBR 17.3+ on Serverless compute (Serverless env v3+ required for VARIANT output support)

**Syntax:** `ai_parse_document(content [, options])`
- `content`: BINARY — document content loaded from `read_files()` or `spark.read.format("binaryFile")`
- `options`: MAP\<STRING, STRING\> (optional) — parsing configuration

**Supported formats:** PDF, JPG/JPEG, PNG, DOCX, PPTX

Returns a VARIANT with pages, elements (text paragraphs, tables, figures, headers, footers), bounding boxes, and error metadata.

**Options:**

| Key | Values | Description |
|-----|--------|-------------|
| `version` | `'2.0'` | Output schema version — always pass this |
| `imageOutputPath` | Volume path | Save rendered page images |
| `descriptionElementTypes` | `''`, `'figure'`, `'*'` | AI-generated descriptions (default: `'*'` for all) |

**Output schema:**

```
document
├── pages[]          -- page id, image_uri
└── elements[]       -- extracted content
    ├── type         -- "text", "table", "figure", etc.
    ├── content      -- extracted text
    ├── bbox         -- bounding box coordinates
    └── description  -- AI-generated description
metadata             -- file info, schema version
error_status[]       -- errors per page (if any)
```

```sql
-- Parse and extract text blocks.
-- The result is a VARIANT { "document": { "pages": [...], "elements": [...] }, "error_status": ..., "metadata": ... }
-- Navigate it with the colon (:) operator — dot notation returns NULL on a VARIANT column.
SELECT
    path,
    concat_ws('\n', transform(parsed:document:elements, e -> e:content::STRING)) AS text_blocks,
    parsed:error_status AS parse_error
FROM (
    SELECT path, ai_parse_document(content, map('version', '2.0')) AS parsed
    FROM read_files('/Volumes/catalog/schema/landing/docs/', format => 'binaryFile')
);

-- Parse with options (image output + descriptions)
SELECT ai_parse_document(
    content,
    map(
        'version', '2.0',
        'imageOutputPath', '/Volumes/catalog/schema/volume/images/',
        'descriptionElementTypes', '*'
    )
) AS parsed
FROM read_files('/Volumes/catalog/schema/volume/invoices/', format => 'binaryFile');
```

```python
from pyspark.sql.functions import expr

df = (
    spark.read.format("binaryFile")
    .load("/Volumes/catalog/schema/landing/docs/")
    .withColumn("parsed", expr("ai_parse_document(content, map('version', '2.0'))"))
    # ai_parse_document returns a VARIANT — navigate with the colon (:) operator, never dot.
    .selectExpr(
        "path",
        "concat_ws('\n', transform(parsed:document:elements, e -> e:content::STRING)) AS text_blocks",
        "parsed:error_status AS parse_error",
    )
    .filter("parse_error IS NULL")
)

# Chain with task-specific functions on the extracted text
df = (
    df.withColumn("summary",  expr("ai_summarize(text_blocks, 50)"))
      .withColumn("entities", expr("ai_extract(text_blocks, '[\"date\", \"amount\", \"vendor\"]')"))
      .withColumn("category", expr("""
          ai_classify(
              text_blocks,
              '["invoice", "contract", "bank_statement"]',
              map('version', '2.0')
          ):response[0]::STRING
      """))
)
df.display()
```

**Limitations:**
- Processing is slow for dense or low-resolution documents
- Suboptimal for non-Latin alphabets and digitally signed PDFs
- Custom models not supported — always uses the built-in parsing model

---

## `ai_prep_search`

**Docs:** https://docs.databricks.com/sql/language-manual/functions/ai_prep_search

**Requires:** DBR 18.2+ on Serverless compute. Beta since April 8, 2026.

**Syntax:** `ai_prep_search(parsed_document [, options])`
- `parsed_document`: VARIANT — the direct output of `ai_parse_document`. Do not pass raw text.
- `options`: MAP\<STRING, STRING\> (optional)

Takes the VARIANT output of `ai_parse_document` and produces a VARIANT array of context-enriched chunks, one per retrievable segment. Each chunk includes surrounding header context, page number, and auto-generated related questions injected alongside the chunk text — producing significantly better vector search recall than naive element concatenation.

**Returns VARIANT** — an array of chunk objects. Use `explode()` or `variant_explode_outer()` to expand to one row per chunk.

**Chunk output shape (per element after explode):**

| Field | Type | Description |
|-------|------|-------------|
| `chunk_to_embed` | STRING | Context-enriched text — **this is the field to embed**, not raw content |
| `chunk_index` | INTEGER | Zero-based position within the document |
| `page_number` | INTEGER | Source page number |
| `element_type` | STRING | Element type (`"text"`, `"table"`, `"figure"`, etc.) |
| `content` | STRING | Raw element content without enrichment |

> **Always embed `chunk_to_embed`, not `content`.** The enrichment (surrounding headers, related questions) is what makes retrieval accurate.

```python
from pyspark.sql.functions import expr, col

# Step 1: Parse
df_parsed = (
    spark.read.format("binaryFile")
    .load("/Volumes/catalog/schema/landing/docs/")
    .withColumn("parsed", expr("ai_parse_document(content, map('version', '2.0'))"))
    .filter("parsed:error_status IS NULL")
    .select("path", "parsed")
)

# Step 2: Prep for search — replaces manual element explode + concat
df_chunks = (
    df_parsed
    .withColumn("chunks_raw", expr("ai_prep_search(parsed)"))
    .withColumn("chunk", expr("explode(chunks_raw)"))
    .select(
        "path",
        col("chunk:chunk_to_embed").alias("chunk_to_embed"),   # context-enriched — embed this
        col("chunk:chunk_index").alias("chunk_index"),
        col("chunk:page_number").alias("page_number"),
        col("chunk:element_type").alias("element_type"),
        col("chunk:content").alias("raw_content"),
    )
)

# Step 3: Write to Delta — enable CDF for Vector Search Delta Sync
(
    df_chunks.write
    .format("delta")
    .option("delta.enableChangeDataFeed", "true")
    .mode("overwrite")
    .saveAsTable("catalog.schema.document_chunks")
)
```

```sql
-- SQL equivalent (useful for DLT tables)
SELECT
    path,
    chunk:chunk_to_embed::STRING  AS chunk_to_embed,
    chunk:chunk_index::INT        AS chunk_index,
    chunk:page_number::INT        AS page_number,
    chunk:element_type::STRING    AS element_type
FROM (
    SELECT
        path,
        explode(ai_prep_search(ai_parse_document(content, map('version', '2.0')))) AS chunk
    FROM read_files('/Volumes/catalog/schema/landing/docs/', format => 'binaryFile')
)
WHERE chunk:chunk_to_embed IS NOT NULL;
```

**Position in the pipeline:**
```
ai_parse_document  →  ai_prep_search  →  embed chunk_to_embed  →  Databricks AI Search index
```

`ai_prep_search` replaces the manual `variant_explode_outer` + `concat_ws` + header-injection approach. In the `cashRecon-search` bundle, use it as the direct source for all three vector indexes (bank statements, invoices, contracts).

**Limitations:**
- Beta — API surface may change before GA
- Requires Serverless compute (DBR 18.2+); not available on Classic clusters
- Only accepts VARIANT output from `ai_parse_document` — not raw strings or other VARIANT shapes
