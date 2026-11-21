# ADR 0001: Dashboard deployment scope — one bundle for the Schufa stack

* **Status:** Accepted
* **Date:** 2026-11-20
* **Owner:** horobaer@gmail.com
* **Related:** bundle `/githubActions/schufa_medallion_pipeline` · dashboard `Schufa Mock Gold Scores` (.lvdash.json)

## Context

The SCHUFA mock stack consists of a producer job, a landing volume, a Lakeflow Spark Declarative Pipeline (events_bronze → events_silver → person_scores gold), and an AI/BI dashboard over the gold tables.

Current state:

* The pipeline is already managed by the Declarative Automation Bundle `schufa_medallion_pipeline` (`databricks.yml` with `include: resources/*.yml`, targets `dev` (default, development mode) and `prod` (production mode)).
* The dashboard exists as a plain `.lvdash.json` workspace file (currently in the `schufaMock` Git repo), **not** as a bundle resource. Its two SQL datasets hardcode `test.schufa_mock.*` table references.
* New requirement: roll the dashboard out across DEV / TEST / PROD.

Question answered here: does the dashboard need to be a bundle resource for that rollout, and should it join the existing pipeline bundle or get its own?

## Decision

1. **The dashboard joins the existing `schufa_medallion_pipeline` bundle** as a `resources.dashboards` entry in its own resource file (e.g. `resources/schufa_gold_scores.dashboard.yml` — already picked up by the bundle's `include: resources/*.yml` pattern), referencing the `.lvdash.json` via `file_path`.
2. **Environment isolation is achieved with bundle variables**, not hardcoded catalogs: the dashboard resource sets `dataset_catalog` / `dataset_schema` (and `warehouse_id`) from per-target variables, so the same dashboard definition resolves to `dev.schufa_mock` in dev, `test.schufa_mock` in test, and `prod.schufa_mock` in prod. The hardcoded `test.schufa_mock` references in the datasets must be removed accordingly.
3. **The whole stack is one deployable unit**: `databricks bundle validate` → `databricks bundle deploy -t <target>` ships pipeline and dashboard together. A `test` target is added alongside the existing `dev` and `prod` targets.
4. **Split criterion (re-evaluate later):** split the dashboard into its own bundle when (a) ownership diverges (BI/analysts vs data engineering), (b) change cadences diverge significantly, or (c) deploy permissions must differ. Until then, one bundle.

## Alternatives considered

* **Separate BI bundle** — cleaner blast radius and independent cadence, but premature for a solo-owned project: it invites drift between pipeline schema changes and the dashboards that read them, and doubles the CI/CD path. Revisit per the split criterion.
* **Git-folder sync only (status quo)** — a Git folder syncs within a single workspace; it cannot parameterize catalog/schema per environment, and the hardcoded `test.*` references would point dev and prod at the same tables. Rejected for multi-environment rollout.
* **Scripted export/import (`databricks lakeview get/create`)** — works, but manual and unreviewed; no variable substitution, no CI discipline. Rejected.

## Consequences

**Positive:**

* Atomic, reviewable deploys — a gold-schema change and the dashboard fix for it ship in one PR and one deploy.
* Per-target catalog/schema/warehouse via variables; same dashboard definition for all environments.
* Single CI/CD path (`validate` → `deploy`) for the whole stack, deployable from the existing GitHub Actions workflow.

**Negative / accepted risks:**

* Review-scope coupling: every chart tweak is deployed through the pipeline bundle's review. For a solo project this is acceptable; the split criterion above triggers extraction if it stops being so.
* Deploy permissions couple dashboard and pipeline. Same mitigation.
* A dashboard-only deploy does **not** restart or disturb the running pipeline (bundle deploys update definitions idempotently), so the runtime risk of combining is limited to review noise and access coupling.

**Follow-ups:**

* Move the `.lvdash.json` into the bundle (e.g. `src/` or alongside `resources/`) and point `file_path` at it.
* Add the `test` target to `databricks.yml` and define the catalog/schema/warehouse variables per target.
* Remove hardcoded `test.schufa_mock` references from the dashboard datasets in favor of the resource-level `dataset_catalog` / `dataset_schema`.
* **Commit and push all changes to GitHub**: the `learn-github-actions.yml` workflow runs `databricks repos update /Workspace/githubActions --branch main` on every push, which syncs this folder from GitHub and overwrites uncommitted local workspace changes.

## References

* Manage dashboards with DABs, incl. `dataset_catalog` / `dataset_schema`: https://docs.databricks.com/aws/en/dev-tools/bundles/examples/
* DAB variables and substitutions: https://docs.databricks.com/aws/en/dev-tools/bundles/variables/
* Modular bundle config via `include`: https://docs.databricks.com/aws/en/dev-tools/bundles/settings/