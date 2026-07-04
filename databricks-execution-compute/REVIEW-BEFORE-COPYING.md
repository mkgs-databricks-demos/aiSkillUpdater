# Review Before Copying: databricks-execution-compute

## Update Applied

- Updated skill metadata to `0.2.0`.
- Confirmed CLI compatibility uses `>= v1.0.0` / `v1.0.0+` wording.
- Updated serverless job environment examples from `client: "4"` to `client: "5"`.
- Added the serverless environment version ladder: v1 through v5, with v5 as the current latest.
- Removed fixed timeout-cap guidance and clarified that task/job timeout uses `timeout_seconds`.
- Updated Python minimum references to Python 3.10+.
- Updated cluster runtime examples to `17.3.x-scala2.12`.

## Reviewer Checks

- Confirm current workspace serverless environment support before copying if distributed to older workspaces.
- Re-run a markdown search for stale serverless client examples, Python versions below 3.10, old runtime strings, and fixed timeout-cap claims.
