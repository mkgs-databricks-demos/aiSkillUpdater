# aiSkillUpdater

A Databricks App to schedule and maintain your Claude Code AI skills — keeping
the Databricks skill library (`~/.databricks/aitools/skills/`) current with
the live Databricks platform, automatically.

## Repository Contents

| Path | What it is |
|------|-----------|
| `docs/PROJECT-PLAN.md` | The design/build plan for the automated aiSkillUpdater app (schedule, audit, PR workflow). **Start here.** |
| `docs/STAGING-AREA.md` | How the current manual staging workflow works (this repo doubles as a review area before deploying skill updates to the live location). |
| `databricks-*/` | One directory per Databricks skill, containing the latest audited/updated `SKILL.md`, `references/`, and a `REVIEW-BEFORE-COPYING.md` change log. |

## Status

This repo currently holds the **28-skill audit snapshot from 2026-07-04**
(see `docs/STAGING-AREA.md` for the audit history table and deploy commands).
The automated scheduling app described in `docs/PROJECT-PLAN.md` has not been
built yet — today, audits are run on demand via Polly
(`/investigate each of the Databricks skills and determine what pieces are
outdated`).

## License

MIT — see `LICENSE`.
