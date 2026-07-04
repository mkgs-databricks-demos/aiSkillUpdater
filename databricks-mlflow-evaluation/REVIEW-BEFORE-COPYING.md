# Review Before Copying — databricks-mlflow-evaluation Skill Update

## Summary of Changes (0.1.0 → 0.2.0)

All files have been updated with the following changes. Review this document before copying files to the canonical skill location.

---

## Universal Change (all files)

- **MLflow version**: All occurrences of `3.1.0` replaced with `3.14.0` throughout every file.

---

## SKILL.md

| Change | Details |
|--------|---------|
| Version bump | `0.1.0` → `0.2.0` in frontmatter |
| Workflow 9 added | New "Workflow 9 — ConversationSimulator" section added after Workflow 8, describing `mlflow.genai.ConversationSimulator` for generating multi-turn eval datasets |
| `@mlflow.test` note added | New Critical API Fact entry: `@mlflow.test` decorator integrates MLflow evaluations into pytest; requires MLflow 3.14.0+ and `pip install mlflow[pytest]` |
| `make_judge()` deprecation note added | New Critical API Fact entry: `make_judge()` is deprecated as of MLflow 3.14.0 — use `@scorer` with `Guidelines` or custom scoring logic instead |
| LoggedModel Lifecycle APIs section added | New section listing all 5 new APIs: `mlflow.set_active_model()`, `mlflow.initialize_logged_model()`, `mlflow.finalize_logged_model()`, `mlflow.search_logged_models()`, `mlflow.create_external_model()` |
| Production Monitoring section added | New section documenting `mlflow.genai.create_monitor()` with example code |

---

## references/patterns-evaluation.md

| Change | Details |
|--------|---------|
| Version replacement | `3.1.0` → `3.14.0` throughout |
| ConversationSimulator section added | New section at end of file with complete code example for `mlflow.genai.ConversationSimulator` including `system_prompt`, `num_turns`, `num_conversations` parameters and notes on MLflow 3.14.0+ requirement |

---

## references/CRITICAL-interfaces.md

| Change | Details |
|--------|---------|
| Version replacement | Version header updated: `MLflow 3.1.0+` → `MLflow 3.14.0+`; installation pip command updated |
| Table of Contents | New entries: `LoggedModel Lifecycle APIs` added |
| `make_judge()` deprecation note | Added `# DEPRECATED in MLflow 3.14.0` comment on the `make_judge` import line in the Judges API section |
| `mlflow.genai.create_monitor()` added | New subsection in Production Monitoring showing `create_monitor()` usage before the existing lower-level register/start pattern |
| LoggedModel Lifecycle APIs section added | Complete new section with code examples for all 5 LoggedModel APIs: `set_active_model`, `initialize_logged_model`, `finalize_logged_model`, `search_logged_models`, `create_external_model` |

---

## references/GOTCHAS.md

| Change | Details |
|--------|---------|
| Version replacement | `3.1.0` → `3.14.0` throughout |
| Table of Contents | Three new entries added: `@mlflow.test Without Required Dependencies`, `Using make_judge() in MLflow 3.14.0+`, `Wrong Production Monitoring — Legacy Setup vs create_monitor()` |
| `@mlflow.test` gotcha added | New full section: "`@mlflow.test` requires MLflow 3.14.0+ and pytest plugin installed (`pip install mlflow[pytest]`)" with wrong/correct code examples |
| `make_judge()` deprecation section added | New full section: "`make_judge()` removed in 3.14.0 — migrate to `@scorer` decorator" with migration examples |
| `mlflow.genai.create_monitor()` section added | New full section: "`mlflow.genai.create_monitor()` replaces the legacy monitor setup in 3.14.0+" with comparison of old vs new approach |
| Summary Checklist | Three new checklist items added corresponding to the new gotchas above |

---

## All Other Reference Files (version replacement only)

The following files were copied with only the `3.1.0` → `3.14.0` version replacement applied. No other content changes were made:

- `references/patterns-context-optimization.md`
- `references/patterns-datasets.md`
- `references/patterns-judge-alignment.md`
- `references/patterns-prompt-optimization.md`
- `references/patterns-scorers.md`
- `references/patterns-trace-analysis.md`
- `references/patterns-trace-ingestion.md`
- `references/user-journeys.md`

Note: Most of these files contain no `3.1.0` version strings, so they are effectively unchanged from the source. The version string `3.9.0` (for UC trace ingestion minimum version) was intentionally left at `3.9.0` where it referred to a specific feature requirement, not the general MLflow version.

---

## Files NOT Changed

- `agents/openai.yaml` — Not a text file requiring version updates; copied as-is via the existing agents directory (not included in this update batch).
- `assets/databricks.png`, `assets/databricks.svg` — Binary/SVG asset files; not applicable.
