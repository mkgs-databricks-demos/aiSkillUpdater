# End-to-End RAG with Databricks AI Search

Build a complete Retrieval-Augmented Generation pipeline: prepare documents, create a vector index, query it, and wire it into an agent.

## CLI Commands Used

| Command | Step |
|---------|------|
| `databricks experimental aitools tools query` | Create source table, insert documents |
| `databricks vector-search-endpoints create-endpoint` | Create compute endpoint |
| `databricks vector-search-indexes create-index` | Create Delta Sync index with managed embeddings |
| `databricks vector-search-indexes sync-index` | Trigger index sync |
| `databricks vector-search-indexes get-index` | Check index status |
| `databricks vector-search-indexes query-index` | Test similarity search |

---

## Step 0: Context-Enriched Chunking with ai_prep_search (Recommended)

> **DBR 18.2+ (Beta since Apr 2026)**: For best retrieval quality, use `ai_prep_search` upstream of the vector index to produce context-enriched chunks. The output `chunk_to_embed` field is what you index.

`ai_prep_search` enriches raw document chunks with surrounding context before embedding, dramatically improving retrieval quality for long documents.

```sql
-- Use ai_prep_search to produce context-enriched chunks
CREATE OR REPLACE TABLE catalog.schema.knowledge_base_chunks AS
SELECT
  doc_id,
  chunk_index,
  chunk_to_embed,   -- This field contains context-enriched text — index this
  chunk_text        -- Original chunk text (for display)
FROM (
  SELECT
    doc_id,
    ai_prep_search(content) AS prep_result
  FROM catalog.schema.raw_documents
)
LATERAL VIEW explode(prep_result.chunks) AS chunk_row
SELECT
  doc_id,
  chunk_row.chunk_index AS chunk_index,
  chunk_row.chunk_to_embed AS chunk_to_embed,
  chunk_row.chunk_text AS chunk_text;
```

Use `chunk_to_embed` as the embedding source column in your index (Step 3 below).

---

## Step 1: Prepare Source Table

The source Delta table needs a primary key column and a text column to embed.

```sql
CREATE TABLE IF NOT EXISTS catalog.schema.knowledge_base (
    doc_id STRING,
    title STRING,
    content STRING,
    category STRING,
    updated_at TIMESTAMP DEFAULT current_timestamp()
);

INSERT INTO catalog.schema.knowledge_base VALUES
('doc-001', 'Getting Started', 'Databricks is a unified analytics platform...', 'overview', current_timestamp()),
('doc-002', 'Unity Catalog', 'Unity Catalog provides centralized governance...', 'governance', current_timestamp()),
('doc-003', 'Delta Lake', 'Delta Lake is an open-source storage layer...', 'storage', current_timestamp());
```

Or via CLI:

```bash
databricks experimental aitools tools query --warehouse WAREHOUSE_ID "
    CREATE TABLE IF NOT EXISTS catalog.schema.knowledge_base (
        doc_id STRING,
        title STRING,
        content STRING,
        category STRING,
        updated_at TIMESTAMP DEFAULT current_timestamp()
    )
"
```

## Step 2: Create AI Search Endpoint

```python
manage_vs_endpoint(
    action="create",
    name="my-rag-endpoint",
    endpoint_type="STORAGE_OPTIMIZED"
)
```

Endpoint creation is asynchronous. Check status:

```python
manage_vs_endpoint(action="get", name="my-rag-endpoint")
# Wait for state: "ONLINE"
```

## Step 3: Create Delta Sync Index

```python
# If using ai_prep_search (recommended, DBR 18.2+), index the chunk_to_embed column
manage_vs_index(
    action="create",
    name="catalog.schema.knowledge_base_index",
    endpoint_name="my-rag-endpoint",
    primary_key="doc_id",
    index_type="DELTA_SYNC",
    delta_sync_index_spec={
        "source_table": "catalog.schema.knowledge_base_chunks",  # output of ai_prep_search
        "embedding_source_columns": [
            {
                "name": "chunk_to_embed",   # Context-enriched field from ai_prep_search
                "embedding_model_endpoint_name": "databricks-qwen3-embedding-0-6b"
            }
        ],
        "pipeline_type": "TRIGGERED",
        "columns_to_sync": ["doc_id", "chunk_index", "chunk_text", "category"]
    }
)

# Without ai_prep_search: index raw content directly
manage_vs_index(
    action="create",
    name="catalog.schema.knowledge_base_index",
    endpoint_name="my-rag-endpoint",
    primary_key="doc_id",
    index_type="DELTA_SYNC",
    delta_sync_index_spec={
        "source_table": "catalog.schema.knowledge_base",
        "embedding_source_columns": [
            {
                "name": "content",
                "embedding_model_endpoint_name": "databricks-qwen3-embedding-0-6b"
            }
        ],
        "pipeline_type": "TRIGGERED",
        "columns_to_sync": ["doc_id", "title", "content", "category"]
    }
)
```

Key decisions:
- **`embedding_source_columns`**: Databricks computes embeddings automatically from the specified column
- **`pipeline_type`**: `TRIGGERED` for manual sync (cheaper), `CONTINUOUS` for auto-sync on table changes
- **`columns_to_sync`**: Only sync columns you need in query results (reduces storage and improves performance)

## Step 4: Sync and Verify

```python
# Trigger initial sync
manage_vs_index(action="sync", index_name="catalog.schema.knowledge_base_index")

# Check status
manage_vs_index(action="get", index_name="catalog.schema.knowledge_base_index")
# Wait for state: "ONLINE"
```

## Step 5: Query the Index

```python
# Semantic search
query_vs_index(
    index_name="catalog.schema.knowledge_base_index",
    columns=["doc_id", "title", "content", "category"],
    query_text="How do I govern my data?",
    num_results=3
)
```

### With Filters

The filter syntax depends on the endpoint type used when creating the index.

```python
# Storage-Optimized endpoint (used in this walkthrough): SQL-like filter syntax
query_vs_index(
    index_name="catalog.schema.knowledge_base_index",
    columns=["doc_id", "title", "content"],
    query_text="How do I govern my data?",
    num_results=3,
    filters="category = 'governance'"
)

# Standard endpoint (if you created a Standard endpoint instead): JSON filters_json
query_vs_index(
    index_name="catalog.schema.my_standard_index",
    columns=["doc_id", "title", "content"],
    query_text="How do I govern my data?",
    num_results=3,
    filters_json='{"category": "governance"}'
)
```

### Hybrid Search (Vector + Keyword)

```python
query_vs_index(
    index_name="catalog.schema.knowledge_base_index",
    columns=["doc_id", "title", "content"],
    query_text="Delta Lake ACID transactions",
    num_results=5,
    query_type="HYBRID"
)
```

---

## Step 6: Use in an Agent

### As a Tool in a ChatAgent

Use `VectorSearchRetrieverTool` to wire the index into an agent deployed on Model Serving:

```python
from databricks.agents import ChatAgent
from databricks.agents.tools import VectorSearchRetrieverTool
from databricks.sdk import WorkspaceClient

# Define the retriever tool
retriever_tool = VectorSearchRetrieverTool(
    index_name="catalog.schema.knowledge_base_index",
    columns=["doc_id", "title", "content"],
    num_results=3,
)

class RAGAgent(ChatAgent):
    def __init__(self):
        self.w = WorkspaceClient()

    def predict(self, messages, context=None):
        query = messages[-1].content

        results = self.w.vector_search_indexes.query_index(
            index_name="catalog.schema.knowledge_base_index",
            columns=["title", "content"],
            query_text=query,
            num_results=3,
        )

        context_docs = "\n\n".join(
            f"**{row[0]}**: {row[1]}"
            for row in results.result.data_array
        )

        response = self.w.serving_endpoints.query(
            name="databricks-meta-llama-3-3-70b-instruct",
            messages=[
                {"role": "system", "content": f"Answer using this context:\n{context_docs}"},
                {"role": "user", "content": query},
            ],
        )

        return {"content": response.choices[0].message.content}
```

---

## Updating the Index

### Add New Documents

```sql
INSERT INTO catalog.schema.knowledge_base VALUES
('doc-004', 'MLflow', 'MLflow is an open-source platform for ML lifecycle...', 'ml', current_timestamp());
```

Then sync:

```python
manage_vs_index(action="sync", index_name="catalog.schema.knowledge_base_index")
```

### Delete Documents

```sql
DELETE FROM catalog.schema.knowledge_base WHERE doc_id = 'doc-001';
```

Then sync — the index automatically handles deletions via Delta change data feed.

---

## Common Issues

| Issue | Solution |
|-------|----------|
| **Index stuck in PROVISIONING** | Endpoint may still be creating. Check `manage_vs_endpoint(action="get")` first |
| **Query returns no results** | Index may not be synced yet. Run `manage_vs_index(action="sync")` and wait for ONLINE state |
| **"Column not found in index"** | Column must be in `columns_to_sync`. Recreate index with the column included |
| **Embeddings not computed** | Ensure `embedding_model_endpoint_name` is a valid serving endpoint |
| **Stale results after table update** | For TRIGGERED pipelines, you must call `manage_vs_index(action="sync")` manually |
| **Filter not working** | Standard endpoints use dict-format filters (`filters_json`), Storage-Optimized use SQL-like string filters (`filters`) |
| **ai_prep_search not available** | Requires DBR 18.2+. Check cluster runtime version. |
