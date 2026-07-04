# Review Before Copying: databricks-lakebase

## Update Applied

- Updated skill metadata to `0.2.0`.
- Updated CLI compatibility to `>= v1.0.0`.
- Removed stale pre-GA status labels from Lakebase, Data API, synced tables, and Lakehouse Sync markdown content.
- Promoted Branching in `SKILL.md` as a first-class feature with copy-on-write branches, point-in-time recovery, TTL, and branch CLI discovery guidance.
- Kept existing `postgres` CLI examples while adding `databricks lakebase branches` discovery because command group rollout may vary by workspace.

## Reviewer Checks

- Confirm the final production CLI command group naming expected by your Databricks distribution.
- Confirm Lakehouse Sync status in your release notes before copying if this skill is distributed to workspaces with feature-gated rollout.
- Re-run a markdown search for stale status labels and old CLI/version strings after copying.
