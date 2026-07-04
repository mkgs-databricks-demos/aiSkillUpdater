---
name: trigger-and-cost-optimization
description: Select and tune triggers for Spark Structured Streaming to balance latency and cost, including Serverless trigger restrictions, availableNow, processing-time triggers, and Real-Time Mode (RTM).
---

# Trigger and Cost Optimization

Choose the trigger based on compute type, latency SLA, and cost target. On Databricks Serverless, trigger support is more constrained than on classic clusters, so validate the trigger before recommending a pattern.

---

## Trigger Decision Table

| Need | Recommended trigger | Compute | Notes |
|---|---|---|---|
| Batch-style incremental processing | `trigger(availableNow=True)` | Serverless or classic | Recommended for Serverless when the stream should process all available data and stop. |
| Sub-second latency | `trigger(processingTime="0 seconds")` or supported continuous RTM trigger | Serverless | Real-Time Mode (RTM); no micro-batch overhead. |
| Low-latency classic streaming | `trigger(processingTime="30 seconds")` or another interval | Classic clusters | Not supported on Serverless. |
| Scheduled ingestion every N minutes/hours | Job schedule + `availableNow=True` | Serverless or classic | Reduces cost by terminating between runs. |
| Always-on classic stream | Processing-time trigger | Classic clusters | Use when serverless trigger limits or libraries require classic. |

---

## Serverless Trigger Rules

Use these rules whenever the user asks for Serverless Structured Streaming:

```python
# Recommended: serverless batch-style streaming
query = (df.writeStream
    .format("delta")
    .option("checkpointLocation", checkpoint_path)
    .trigger(availableNow=True)
    .toTable("catalog.schema.table"))

# RTM / lowest-latency serverless streaming
query = (df.writeStream
    .format("delta")
    .option("checkpointLocation", checkpoint_path)
    .trigger(processingTime="0 seconds")
    .toTable("catalog.schema.table"))
```

Unsupported on Serverless:

```python
# Unsupported on Serverless
.trigger(processingTime="30 seconds")

# Unsupported on Serverless
.trigger(continuous="1 second")
```

`foreachBatch` **is supported** on Serverless. Do not reject Serverless just because a pipeline uses `foreachBatch`; validate the rest of the query plan, sink, dependencies, and trigger.

---

## Real-Time Mode (RTM)

Real-Time Mode is Databricks' lowest-latency Structured Streaming mode for workloads that need sub-second processing. It removes micro-batch scheduling overhead and is intended for operational analytics, monitoring, real-time alerting, and low-latency event processing.

### Requirements

- Serverless compute.
- Trigger configured for RTM, such as `trigger(processingTime="0 seconds")` or the supported continuous trigger form in the target workspace/runtime.
- Query plan compatible with RTM limitations.

### RTM Limitations

| Limitation | Guidance |
|---|---|
| `transformWithStateInPandas` is not supported | Use row-based `transformWithState`. |
| Self-union is not supported | Refactor the query to avoid unioning a stream with itself. |
| Union with a batch source is not supported | Materialize the batch side separately or use a stream-static join pattern where supported. |
| Some sinks/options may have runtime-specific constraints | Validate against the current Databricks runtime docs and run a small end-to-end test. |

### RTM Example

```python
from pyspark.sql.functions import col

query = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", bootstrap_servers)
    .option("subscribe", "events")
    .load()
    .selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)", "timestamp")
    .filter(col("value").isNotNull())
    .writeStream
    .format("delta")
    .option("checkpointLocation", "/Volumes/main/ops/checkpoints/events_rtm")
    .trigger(processingTime="0 seconds")
    .toTable("main.ops.events_realtime"))
```

### RTM Stateful Processing

```python
# Use row-based transformWithState for RTM-compatible stateful logic.
# Do not use transformWithStateInPandas in RTM.
stateful = input_stream.transformWithState(
    statefulProcessor=processor,
    timeMode="EventTime",
    outputMode="Append"
)
```

---

## availableNow for Cost Control

`availableNow=True` processes all currently available data and then stops. Pair it with a Databricks Job schedule for predictable cost and simple retry semantics.

```python
query = (spark.readStream
    .table("main.bronze.orders_raw")
    .writeStream
    .foreachBatch(upsert_orders)
    .option("checkpointLocation", "/Volumes/main/checkpoints/orders_upsert")
    .trigger(availableNow=True)
    .start())
```

Use `availableNow` when:

- The SLA is minutes or hours rather than sub-second.
- The source supports catching up from checkpointed offsets.
- You want compute to shut down after each run.
- The pipeline uses `foreachBatch` for MERGE/upsert logic.

---

## Processing-Time Triggers on Classic Compute

Processing-time triggers remain useful on classic clusters when the stream must run continuously and Serverless trigger restrictions do not fit.

```python
query = (df.writeStream
    .format("delta")
    .option("checkpointLocation", checkpoint_path)
    .trigger(processingTime="5 minutes")
    .toTable("catalog.schema.table"))
```

Tuning guidance:

- Choose a trigger interval comfortably larger than typical batch processing time.
- Increase the interval to reduce cost when latency SLA allows.
- Decrease the interval only after confirming the stream can keep up.
- Monitor input rows per second, processed rows per second, batch duration, and backlog.

---

## Cost Patterns

| Pattern | Cost profile | Latency | Use when |
|---|---|---|---|
| Job schedule + `availableNow=True` | Lowest | Schedule interval + processing time | Batch-style streaming, incremental ETL, serverless workloads. |
| RTM on Serverless | Highest | Sub-second | Real-time alerts and operational analytics. |
| Always-on classic processing-time stream | Medium to high | Trigger interval + processing time | Classic-only dependencies or unsupported serverless triggers. |
| Multiple streams on shared classic cluster | Lower per stream | Depends on trigger | Several steady streams with compatible libraries and isolation needs. |

---

## Common Issues

| Issue | Cause | Fix |
|---|---|---|
| Serverless stream fails with `processingTime="30 seconds"` | Processing-time intervals other than RTM zero-second mode are unsupported on Serverless | Use `availableNow=True` or RTM `processingTime="0 seconds"`. |
| Serverless stream fails with `continuous="1 second"` | Continuous trigger form is unsupported on Serverless | Use supported RTM trigger guidance or classic compute. |
| RTM stateful query fails | Uses `transformWithStateInPandas` | Replace with row-based `transformWithState`. |
| RTM union query fails | Self-union or union with a batch source | Refactor query topology. |
| Costs are high | Always-on stream for batch-style SLA | Switch to scheduled `availableNow=True`. |
| `foreachBatch` rejected based on stale guidance | Current guidance says `foreachBatch` is supported on Serverless | Inspect the actual error and validate the rest of the query plan. |

---

## Production Checklist

- [ ] Trigger is valid for the selected compute type.
- [ ] Serverless batch-style streams use `availableNow=True`.
- [ ] RTM streams use Serverless and an RTM-compatible trigger.
- [ ] RTM stateful code avoids `transformWithStateInPandas`.
- [ ] RTM query plan avoids self-union and union with batch sources.
- [ ] `foreachBatch` pipelines are evaluated normally for Serverless compatibility.
- [ ] Checkpoint path is stable and unique per stream.
- [ ] Job/task timeout, retries, and schedule match the SLA.
- [ ] Cost monitoring tags identify stream name, environment, and owner.
