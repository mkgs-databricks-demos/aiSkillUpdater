# Databricks AI Search Index Types

## Comparison Matrix

| Feature | Delta Sync (Managed) | Delta Sync (Self-Managed) | Direct Access | Full-Text Search (Beta) |
|---------|---------------------|---------------------------|---------------|------------------------|
| **Embeddings** | Databricks computes | You provide | You provide | None (BM25 keyword) |
| **Sync** | Auto from Delta | Auto from Delta | Manual CRUD | Auto from Delta |
| **Setup** | Easiest | Medium | Most control | Easy (no embeddings) |
| **Source** | Delta table + text | Delta table + vectors | API calls | Delta table + text |
| **Endpoint** | Standard or Storage-Optimized | Standard or Storage-Optimized | Standard or Storage-Optimized | Storage-Optimized only |
| **Best for** | Quick start, RAG | Custom models | Real-time apps | Keyword search, no embeddings |

## Delta Sync with Managed Embeddings

Databricks automatically computes embeddings from your text column.

### Requirements

- Source Delta table with:
  - Primary key column (unique identifier)
  - Text column (content to embed)
- Embedding model endpoint (or use built-in)

### Create Index

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

index = w.vector_search_indexes.create_index(
    name="catalog.schema.docs_index",
    endpoint_name="my-vs-endpoint",
    primary_key="doc_id",
    index_type="DELTA_SYNC",
    delta_sync_index_spec={
        "source_table": "catalog.schema.documents",
        "embedding_source_columns": [
            {
                "name": "content",
                "embedding_model_endpoint_name": "databricks-qwen3-embedding-0-6b"
            }
        ],
        "pipeline_type": "TRIGGERED",  # or "CONTINUOUS"
        "columns_to_sync": ["doc_id", "content", "title", "category"]
    }
)
```

### Pipeline Types

| Type | Behavior | Cost | Use Case |
|------|----------|------|----------|
| `TRIGGERED` | Manual sync via API | Lower | Batch updates |
| `CONTINUOUS` | Auto-sync on changes | Higher | Real-time sync |

### Source Table Example

```sql
CREATE TABLE catalog.schema.documents (
    doc_id STRING,
    title STRING,
    content STRING,  -- Text to embed
    category STRING,
    created_at TIMESTAMP
);
```

## Delta Sync with Self-Managed Embeddings

You pre-compute embeddings and store them in the source table.

### Requirements

- Source Delta table with:
  - Primary key column
  - Embedding vector column (array of floats)

### Create Index

```python
index = w.vector_search_indexes.create_index(
    name="catalog.schema.custom_index",
    endpoint_name="my-vs-endpoint",
    primary_key="id",
    index_type="DELTA_SYNC",
    delta_sync_index_spec={
        "source_table": "catalog.schema.embedded_docs",
        "embedding_vector_columns": [
            {
                "name": "embedding",
                "embedding_dimension": 1024
            }
        ],
        "pipeline_type": "TRIGGERED"
    }
)
```

### Compute Embeddings

```python
from databricks.sdk import WorkspaceClient
import pandas as pd

w = WorkspaceClient()

def get_embeddings(texts: list[str]) -> list[list[float]]:
    """Call embedding endpoint for texts."""
    response = w.serving_endpoints.query(
        name="databricks-qwen3-embedding-0-6b",
        input=texts
    )
    return [item.embedding for item in response.data]

# Add embeddings to your data
df = spark.table("catalog.schema.documents").toPandas()
df["embedding"] = get_embeddings(df["content"].tolist())

# Write back to Delta
spark.createDataFrame(df).write.mode("overwrite").saveAsTable(
    "catalog.schema.embedded_docs"
)
```

### Source Table Example

```sql
CREATE TABLE catalog.schema.embedded_docs (
    id STRING,
    content STRING,
    embedding ARRAY<FLOAT>,  -- Pre-computed embedding
    metadata STRING
);
```

## Direct Access Index

Full control over vector data via CRUD API. No Delta table sync.

### Requirements

- Define schema upfront
- Manage upsert/delete operations yourself

### Create Index

```python
import json

index = w.vector_search_indexes.create_index(
    name="catalog.schema.realtime_index",
    endpoint_name="my-vs-endpoint",
    primary_key="id",
    index_type="DIRECT_ACCESS",
    direct_access_index_spec={
        "embedding_vector_columns": [
            {"name": "embedding", "embedding_dimension": 1024}
        ],
        "schema_json": json.dumps({
            "id": "string",
            "text": "string",
            "embedding": "array<float>",
            "category": "string",
            "score": "float"
        })
    }
)
```

### Upsert Data

```python
import json

# Insert or update vectors
w.vector_search_indexes.upsert_data_vector_index(
    index_name="catalog.schema.realtime_index",
    inputs_json=json.dumps([
        {
            "id": "doc-001",
            "text": "Machine learning basics",
            "embedding": [0.1, 0.2, 0.3, ...],  # 1024 floats
            "category": "ml",
            "score": 0.95
        },
        {
            "id": "doc-002",
            "text": "Deep learning overview",
            "embedding": [0.4, 0.5, 0.6, ...],
            "category": "dl",
            "score": 0.88
        }
    ])
)
```

### Delete Data

```python
w.vector_search_indexes.delete_data_vector_index(
    index_name="catalog.schema.realtime_index",
    primary_keys=["doc-001", "doc-002"]
)
```

### Attach Embedding Model (Optional)

For Direct Access with text queries:

```python
# Create index with embedding model for query-time embedding
index = w.vector_search_indexes.create_index(
    name="catalog.schema.hybrid_index",
    endpoint_name="my-vs-endpoint",
    primary_key="id",
    index_type="DIRECT_ACCESS",
    direct_access_index_spec={
        "embedding_vector_columns": [
            {"name": "embedding", "embedding_dimension": 1024}
        ],
        "embedding_model_endpoint_name": "databricks-qwen3-embedding-0-6b",  # For query_text
        "schema_json": json.dumps({...})
    }
)
```

## Full-Text Search Index (Beta)

Full-Text Search indexes use BM25 keyword scoring without any vector embeddings. They are ideal for pure keyword search use cases where semantic understanding is not needed.

### Requirements

- **Storage-Optimized endpoint** — Full-Text Search indexes require `endpoint_type='STORAGE_OPTIMIZED'`
- Source Delta table with:
  - Primary key column
  - Text column(s) to index

### Create Endpoint and Index

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Step 1: Create Storage-Optimized endpoint (required)
endpoint = w.vector_search_endpoints.create_endpoint(
    name="my-fts-endpoint",
    endpoint_type="STORAGE_OPTIMIZED"
)

# Step 2: Create Full-Text Search index
index = w.vector_search_indexes.create_index(
    name="catalog.schema.fts_index",
    endpoint_name="my-fts-endpoint",
    primary_key="id",
    index_type="DELTA_SYNC",
    delta_sync_index_spec={
        "source_table": "catalog.schema.documents",
        "full_text_search_columns": [
            {"name": "content"}
        ],
        "pipeline_type": "TRIGGERED",
        "columns_to_sync": ["id", "content", "title", "category"]
    }
)
```

### Query a Full-Text Search Index

```python
from databricks.ai_search.client import AISearchClient

client = AISearchClient()
index_client = client.get_index(
    endpoint_name="my-fts-endpoint",
    index_name="catalog.schema.fts_index"
)

# BM25 keyword search
results = index_client.similarity_search(
    query_text="executor memory error spark",
    columns=["id", "content", "title"],
    num_results=10,
    query_type="FULL_TEXT"
)
```

### When to Use Full-Text Search

- Pure keyword matching without semantic similarity
- Exact term or phrase matching
- Cases where embedding models are unavailable or too costly
- Supplement to vector search via hybrid pipelines

### Limitations

- No semantic understanding — only keyword matching
- Maximum 200 results per query
- Requires Storage-Optimized endpoint (higher latency than Standard)

## Choosing the Right Type

```
Start here:
│
├─ Do you need keyword search only (no embeddings)?
│   └─ Yes → Full-Text Search Index (Beta) — requires Storage-Optimized endpoint
│
├─ Do you have pre-computed embeddings?
│   ├─ Yes → Do you want auto-sync from Delta?
│   │         ├─ Yes → Delta Sync (Self-Managed)
│   │         └─ No  → Direct Access
│   │
│   └─ No → Delta Sync (Managed Embeddings)
│
└─ Do you need real-time updates (<1 sec)?
    ├─ Yes → Direct Access
    └─ No  → Delta Sync (any type)
```

## Endpoint Selection

After choosing index type, choose endpoint:

| Scenario | Endpoint Type |
|----------|---------------|
| Need <100ms latency | Standard |
| >100M vectors | Storage-Optimized |
| Cost-sensitive | Storage-Optimized |
| Full-Text Search index | Storage-Optimized (required) |
| Default choice | Storage-Optimized |
