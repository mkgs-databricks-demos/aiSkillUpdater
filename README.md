# Databricks Skill Updates

A recurring maintenance project for keeping the Databricks Claude agent skills
current with the live Databricks platform.

## What This Is

This directory contains updated versions of the Databricks skills installed at:

```
~/.databricks/aitools/skills/
```

Each subdirectory here mirrors the name of the corresponding live skill directory.
Updated files are staged here for review **before** being copied to the live location.

## Directory Structure

```
~/databricks-skill-updates/
├── README.md                          ← this file
├── databricks-ai-functions/
│   ├── REVIEW-BEFORE-COPYING.md       ← change log for this skill
│   ├── SKILL.md                       ← updated skill entry point
│   └── references/                    ← updated reference files
├── databricks-vector-search/
│   └── ...
└── ... (one directory per skill)
```

Every skill directory contains a `REVIEW-BEFORE-COPYING.md` that lists:
- Every file that was changed, with before/after details
- Files that were copied unchanged
- Any flags or caveats to check before deploying

## Audit History

| Date | Audited By | Skills Updated | Notes |
|------|-----------|----------------|-------|
| 2026-07-04 | Polly + claude_code / codex / opencode / pi | All 28 skills | Full initial audit against live docs.databricks.com |

## How to Deploy a Single Skill

```bash
skill=databricks-ai-functions   # change to target skill name

cp -r ~/databricks-skill-updates/$skill/* \
      ~/.databricks/aitools/skills/$skill/
```

## How to Deploy All Skills at Once

```bash
for skill in ~/databricks-skill-updates/databricks-*/; do
  name=$(basename "$skill")
  cp -r "$skill"* ~/.databricks/aitools/skills/"$name"/
done
```

## How to Run a Future Audit Cycle

1. Open a session with Polly and say:
   > `/investigate each of the Databricks skills and determine what pieces are outdated. Come up with a plan to update them.`
2. Polly dispatches parallel sub-agents to read each SKILL.md and cross-check against live `docs.databricks.com`.
3. A master audit report is produced (like `SESSION-00B-skills-audit-<date>.md`).
4. Run the updates:
   > `Update all skills in the same way — write updated Markdown to ~/databricks-skill-updates/`
5. Review each skill's `REVIEW-BEFORE-COPYING.md`.
6. Deploy with the copy commands above.

## Recommended Audit Cadence

| Trigger | Action |
|---------|--------|
| Every **Databricks Runtime major release** (e.g. DBR 19.x) | Full audit of all 28 skills |
| New **AI function** announced | Targeted audit of `databricks-ai-functions` + any skill that calls AI functions |
| After installing a **new skill** | Audit that skill immediately to establish a baseline |
| Any time a skill produces **wrong code** in the field | File-level targeted update for that skill |

## Skills Inventory

28 skills total as of 2026-07-04:

| Skill | Priority | Last Updated |
|-------|----------|--------------|
| databricks-ai-functions | P0 | 2026-07-04 |
| databricks-vector-search | P0 | 2026-07-04 |
| databricks-model-serving | P0 | 2026-07-04 |
| databricks-iceberg | P0 | 2026-07-04 |
| databricks-ml-training | P0 | 2026-07-04 |
| databricks-lakebase | P1 | 2026-07-04 |
| databricks-lakeflow-connect | P1 | 2026-07-04 |
| databricks-execution-compute | P1 | 2026-07-04 |
| databricks-spark-structured-streaming | P1 | 2026-07-04 |
| databricks-unity-catalog | P1 | 2026-07-04 |
| databricks-mlflow-evaluation | P1 | 2026-07-04 |
| databricks-metric-views | P1 | 2026-07-04 |
| databricks-zerobus-ingest | P1 | 2026-07-04 |
| databricks-pipelines | P1 | 2026-07-04 |
| databricks-unstructured-pdf-generation | P1 | 2026-07-04 |
| databricks-synthetic-data-gen | P1 | 2026-07-04 |
| databricks-agent-bricks | P2 | 2026-07-04 |
| databricks-aibi-dashboards | P2 | 2026-07-04 |
| databricks-app-design | P2 | 2026-07-04 |
| databricks-apps | P2 | 2026-07-04 |
| databricks-apps-python | P2 | 2026-07-04 |
| databricks-core | P2 | 2026-07-04 |
| databricks-dabs | P2 | 2026-07-04 |
| databricks-dbsql | P2 | 2026-07-04 |
| databricks-docs | P2 | 2026-07-04 |
| databricks-jobs | P2 | 2026-07-04 |
| databricks-python-sdk | P2 | 2026-07-04 |
| databricks-serverless-migration | P2 | 2026-07-04 |
