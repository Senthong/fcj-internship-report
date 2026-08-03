---
title: "Week 6 Worklog"
date: 2026-07-06
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6 Objectives:

* Orchestrate the full pipeline (ingestion → Redshift load → dbt staging → quality gate → dbt mart → tests → notify) with a single Apache Airflow DAG.
* Implement the quality gate: mart models must not run if staging tests fail.
* Configure retries with exponential backoff and `max_active_runs=1` to avoid overlapping runs.
* Learn Airflow's core concepts (DAG, Operator, Scheduler, Executor) well enough to justify each design decision in the report.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                                   | Start Date | Completion Date | Reference Material                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | ----------------------------------------- |
| 2   | - Study Airflow core concepts: DAG, Operator (Python/Bash), Scheduler, Executor (Local/Celery/Kubernetes), XCom, retry & SLA                                                                            | 07/06/2026 | 07/06/2026      | Apache Airflow docs |
| 3   | - Install Airflow 2.8.1 locally (initially via pip, to iterate faster than rebuilding Docker each time) <br> - Write `DEFAULT_ARGS` with 2 retries, exponential backoff (5 min → max 30 min)            | 07/07/2026 | 07/07/2026      | |
| 4   | - Write `ingest_olist_to_s3` and `load_staging_to_redshift` as `PythonOperator` tasks, wired to the Week 3–4 functions <br> - Write `on_failure_callback=notify_failure` for detailed failure logging   | 07/08/2026 | 07/08/2026      | |
| 5   | - Write 4 `BashOperator` tasks calling the dbt CLI (`dbt run --select staging`, `dbt test --select staging`, `dbt run --select mart`, `dbt test --select mart`), passing Redshift credentials via `env` | 07/09/2026 | 07/09/2026      | |
| 6   | - Chain all 7 tasks linearly with `>>`, placing `dbt_test_staging` as the quality gate before `dbt_run_mart` <br> - Set `schedule_interval="0 2 * * *"`, `max_active_runs=1`, `catchup=False` <br> - **Test:** trigger a manual DAG run end-to-end, then force a staging test to fail and confirm `dbt_run_mart` never executes | 07/10/2026 | 07/11/2026 | |


### Week 6 Achievements:

* Completed `airflow/dags/ecom_full_pipeline_dag.py`: a 7-task linear DAG — `ingest_olist_to_s3 → load_staging_to_redshift → dbt_run_staging → dbt_test_staging → dbt_run_mart → dbt_test_mart → notify_success`.

* Verified the quality gate works as designed: intentionally introduced bad data to fail a `dbt_test_staging` assertion, confirmed the DAG halts there and `dbt_run_mart` never triggers — so no mart table can ever be refreshed with unverified staging data.

* Configured and tested retry behavior: simulated a transient Redshift connection failure, watched Airflow retry with 5-minute exponential backoff (capped at 30 minutes) instead of failing immediately — appropriate for issues like network blips or throttling.

* Understood why `max_active_runs=1` matters: without it, a slow-running DAG instance could still be executing while the next day's scheduled run starts, both writing to the same `staging` schema at once.

* Learned the practical difference between `PythonOperator` (direct function calls, good for the ingestion/load steps that already exist as importable Python functions) and `BashOperator` (good for driving the dbt CLI without reimplementing dbt's own execution logic in Python).

* Ran the DAG on a schedule for several consecutive (backfilled) days to confirm idempotency holds end-to-end: repeated runs for the same logical date didn't duplicate rows in Redshift or in the mart tables.

* First full green run of the entire pipeline, S3 to mart tables, fully automated and unattended — a meaningful milestone before moving on to infrastructure-as-code.
