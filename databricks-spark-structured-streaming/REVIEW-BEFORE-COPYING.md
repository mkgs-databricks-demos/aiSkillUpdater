# Review Before Copying: databricks-spark-structured-streaming

## Update Applied

- Updated skill metadata to `0.2.0`.
- Changed the `SKILL.md` quick start trigger from `processingTime="30 seconds"` to `availableNow=True` for serverless batch-style streaming.
- Added a comprehensive RTM section to `SKILL.md`.
- Added RTM to the core patterns table.
- Replaced trigger/cost reference with current Serverless and RTM guidance.
- Documented Serverless restrictions: `processingTime="30 seconds"` unsupported and `continuous="1 second"` unsupported.
- Documented `availableNow=True` as recommended for serverless batch-style streaming.
- Corrected foreachBatch guidance: `foreachBatch` is supported on Serverless.
- Documented RTM limitations: `transformWithStateInPandas` unsupported, self-union unsupported, union with batch source unsupported.

## Reviewer Checks

- Confirm the exact RTM trigger syntax supported by the target workspace/runtime before copying.
- Re-run a markdown search for stale RTM trigger examples and invalid Serverless trigger recommendations after copying.
