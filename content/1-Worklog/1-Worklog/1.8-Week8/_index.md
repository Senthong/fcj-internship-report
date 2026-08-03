---
title: "Week 8 Worklog"
date: 2026-07-20
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Week 8 Objectives:

* Set up a GitHub Actions CI pipeline: lint Python, compile dbt models, validate Terraform — all on every push/PR.
* Run a full end-to-end regression test of the pipeline (S3 → Redshift → dbt → mart) to confirm nothing broke while wiring everything together.
* Evaluate the system honestly against the original 6 objectives, and document strengths and limitations for the report.
* Finalize the graduation report: write-up, diagrams, code listings, and the appendix with the full DAG source.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                                   | Start Date | Completion Date | Reference Material                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | ----------------------------------------- |
| 2   | - Write `.github/workflows/ci.yml`: `lint-python` job (`black --check`, `flake8`) <br> - Write `dbt-compile` job (`dbt deps && dbt compile --target dev` against a dummy host, no live DB needed)         | 07/20/2026 | 07/20/2026      | GitHub Actions docs |
| 3   | - Write `terraform-validate` job (`terraform init -backend=false`, `terraform validate`, `terraform fmt -check`) <br> - **Test:** open a PR with an intentional lint error and a bad Terraform block, confirm CI fails on both | 07/21/2026 | 07/21/2026      | |
| 4   | - Run a full end-to-end regression: fresh `terraform apply` → Docker Compose up → trigger `ecom_full_pipeline_dag` → verify all 4 mart tables populate correctly and all 46 dbt tests pass                | 07/22/2026 | 07/22/2026      | |
| 5   | - Write Chapter 5 (Results & Evaluation): tally final numbers (11 dbt models, 46 tests, 7-task DAG, 4 Terraform resource groups, 3 CI jobs) against the original 6 objectives <br> - Honestly document limitations (no incremental models, LocalExecutor only, no active alerting, no CD, no unit tests, no BI layer) | 07/23/2026 | 07/23/2026      | |
| 6   | - Write Chapter 6 (Conclusion & Future Work) <br> - Assemble the full report: theory chapter, architecture diagrams, code listings, references, and the DAG appendix <br> - Push final code to the public GitHub repo and do a last read-through of the whole report | 07/24/2026 | 07/25/2026 | |


### Week 8 Achievements:

* Completed `.github/workflows/ci.yml` with 3 independent jobs running in parallel on every push/PR to `main`/`develop`: `lint-python`, `dbt-compile`, `terraform-validate`.

* Confirmed `dbt compile` is enough to catch most SQL/Jinja syntax errors and broken `ref()`/`source()` references without needing a live Redshift connection — kept the CI pipeline fast and free of infrastructure cost.

* Verified CI actually catches problems: deliberately committed a badly formatted Python file and a Terraform block with a syntax error, confirmed both `lint-python` and `terraform-validate` failed the PR check as expected.

* Ran a complete end-to-end regression test from a clean slate (`terraform destroy` → `apply` → Docker Compose → DAG trigger) and confirmed the whole pipeline still worked after 7 weeks of incremental changes — no regressions in idempotency, the quality gate, or transaction rollback behavior.

* Tallied the final project scope for the report: 11 dbt models (7 staging + 4 mart), 46 data quality tests, a 7-task Airflow DAG, 4 Terraform-managed AWS resource groups, and a 3-job CI pipeline.

* Wrote an honest limitations section: mart tables are still full-refresh (not incremental), Airflow runs on `LocalExecutor` only, failure notifications are log-only (no Slack/email), there's no automated Continuous Deployment step, no `pytest` unit tests for the ingestion scripts, and no BI/dashboard layer — all explicitly scoped as future work rather than hidden.

* Finalized and proofread the full graduation report (6 chapters + appendix with the complete DAG source), and pushed the final version of all code to the public repository referenced in the report's cam đoan (declaration of originality) section.
