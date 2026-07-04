---
name: databricks-spark-structured-streaming
description: "Comprehensive guide to Spark Structured Streaming for production workloads. Use when building streaming pipelines, working with Kafka ingestion, implementing Real-Time Mode (RTM), configuring triggers (processingTime, availableNow), handling stateful operations with watermarks, optimizing checkpoints, performing stream-stream or stream-static joins, writing to multiple sinks, or tuning streaming cost and performance."
compatibility: Requires databricks CLI (>= v1.0.0)
metadata:
  version: "0.2.0"
parent: databricks-core
---

# Spark Structured Streaming

Production-ready streaming pipelines with Spark Structured Streaming. This skill provides navigation to detailed patterns and best practices.

## Quick Start

```python
from pyspark.sql.functions import col, from_json

# Basic Kafka to Delta streaming
df = (spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "topic")
    .load()
    .select(from_json(col("value").cast("string"), schema).alias("data"))
    .select("data.*")
)

query = (df.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/Volumes/catalog/checkpoints/stream")
    # Recommended for serverless batch-style streaming: process all available data, then stop.
    .trigger(availableNow=True)
    .start("/delta/target_table"))
```

## Real-Time Mode (RTM)

Use Real-Time Mode when the user needs sub-second streaming latency and the workload can run on Databricks serverless streaming. RTM removes micro-batch scheduling overhead and is designed for low-latency event processing, alerting, and operational analytics.

Requirements and trigger guidance:

- **Compute:** Serverless compute is required.
- **Trigger:** Use `trigger(processingTime="0 seconds")` or the supported continuous trigger form for RTM workloads.
- **Stateful APIs:** `transformWithStateInPandas` is **not supported** in RTM. Use row-based `transformWithState` instead.
- **Union limits:** Self-union is not supported, and union with a batch source is not supported.
- **Serverless batch-style streaming:** Use `trigger(availableNow=True)` when the workload should process all available data and stop.

Serverless trigger restrictions to remember:

```python
# Recommended for serverless batch-style streaming
.trigger(availableNow=True)

# RTM / lowest-latency serverless streaming
.trigger(processingTime="0 seconds")

# Unsupported on Serverless
.trigger(processingTime="30 seconds")
.trigger(continuous="1 second")
```

`foreachBatch` is supported on Serverless. Do not tell users to avoid Serverless solely because their pipeline uses `foreachBatch`.

## Core Patterns

| Pattern | Description | Reference |
|---------|-------------|-----------|
| **Kafka Streaming** | Kafka to Delta, Kafka to Kafka, Real-Time Mode | See [references/kafka-streaming.md](references/kafka-streaming.md) |
| **Real-Time Mode (RTM)** | Sub-second latency on serverless with no micro-batch overhead; use `processingTime='0 seconds'` or continuous trigger; avoid `transformWithStateInPandas` | See [references/trigger-and-cost-optimization.md](references/trigger-and-cost-optimization.md) |
| **Stream Joins** | Stream-stream joins, stream-static joins | See [references/stream-stream-joins.md](references/stream-stream-joins.md), [references/stream-static-joins.md](references/stream-static-joins.md) |
| **Multi-Sink Writes** | Write to multiple tables, parallel merges | See [references/multi-sink-writes.md](references/multi-sink-writes.md) |
| **Merge Operations** | MERGE performance, parallel merges, optimizations | See [references/merge-operations.md](references/merge-operations.md) |

## Configuration

| Topic | Description | Reference |
|-------|-------------|-----------|
| **Checkpoints** | Checkpoint management and best practices | See [references/checkpoint-best-practices.md](references/checkpoint-best-practices.md) |
| **Stateful Operations** | Watermarks, state stores, RocksDB configuration | See [references/stateful-operations.md](references/stateful-operations.md) |
| **Trigger & Cost** | Trigger selection, cost optimization, RTM | See [references/trigger-and-cost-optimization.md](references/trigger-and-cost-optimization.md) |

## Best Practices

| Topic | Description | Reference |
|-------|-------------|-----------|
| **Production Checklist** | Comprehensive best practices | See [references/streaming-best-practices.md](references/streaming-best-practices.md) |

## Production Checklist

- [ ] Checkpoint location is persistent (UC volumes, not DBFS)
- [ ] Unique checkpoint per stream
- [ ] Fixed-size cluster (no autoscaling for streaming)
- [ ] Monitoring configured (input rate, lag, batch duration)
- [ ] Exactly-once verified (txnVersion/txnAppId)
- [ ] Watermark configured for stateful operations
- [ ] Left joins for stream-static (not inner)
