# Build Plans

This folder breaks `docs/PROJECT-PLAN.md` into **self-contained, buildable
build plans** — one per project phase. Each file is written to be handed
to a coding agent (human-supervised) as its complete brief for that phase:
objective, prerequisites, bundle placement, a numbered task breakdown, data
contracts, an explicit fan-out map (what can run in parallel vs. what must
be sequential — both *within* the plan and *across* plans), a validation
checklist, and acceptance criteria.

Every build plan assumes the reader has **not** read `PROJECT-PLAN.md` —
it restates whatever context it needs — but links back to the relevant
`PROJECT-PLAN.md` section(s) for the full design rationale and history of
decisions, rather than re-litigating *why* in the build plan itself.

Each plan was drafted from the locked decisions in `PROJECT-PLAN.md` (all
open questions resolved as of this writing) and then **cross-reviewed** by
an independent sub-agent lens before being marked ready.

## Status

| # | Build plan | Phase(s) covered | Status |
|---|---------------|-------------------|--------|
| 00 | [`00-workspace-setup-and-repo-configuration.md`](00-workspace-setup-and-repo-configuration.md) | Phase 0 | Cross-reviewed (codex + pi) |
| 01 | [`01-rss-ingestion-and-research-agent.md`](01-rss-ingestion-and-research-agent.md) | Phase 2 | Cross-reviewed (codex + pi) |
| 02 | [`02-grounding-corpus.md`](02-grounding-corpus.md) | Phase 1 | Cross-reviewed (codex + pi) |
| 03 | [`03-findings-ingestion-pipeline.md`](03-findings-ingestion-pipeline.md) | Phase 2b | Cross-reviewed (codex + pi) |
| 04 | `04-review-app.md` | Phase 3 | Not started |
| 05 | `05-local-installer.md` | Phase 5 | Not started |
| 06 | `06-cli-drift-detection.md` | Phase 4 | Not started |
| 07 | `07-guardrails-and-observability.md` | Phase 7 | Not started |
| 08 | `08-knowledge-base-search-and-mcp-server.md` | Phase 6 (stretch) | Not started |

Ordering here follows `PROJECT-PLAN.md` §4's suggested build order, not the
phase numbers themselves (Phase 2 before Phase 1, etc.) — see that section
for the rationale.
