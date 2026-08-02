---
title: "Airflow Orchestration & Data Quality Testing"
date: 2026-07-24
weight: 3
chapter: false
pre: " <b> 5.4.3 </b> "
---

Sections 5.4.1 and 5.4.2 described the individual components of the data pipeline, including the ingestion scripts, Staging Models, and Data Mart models. This section explains how these components are orchestrated into a fully automated workflow that executes on a fixed schedule and can automatically stop whenever data quality issues are detected. This orchestration is implemented through the project's single Directed Acyclic Graph (DAG), `ecom_full_pipeline_dag`, defined in `airflow/dags/ecom_full_pipeline_dag.py`.

## 1. DAG Structure and Task Sequence

The DAG is scheduled to run automatically once per day at **02:00 UTC** (09:00 Vietnam Time). It is configured **not** to perform historical backfills (`catchup=False`) when first deployed, and it allows only one active execution at a time (`max_active_runs=1`), preventing delayed pipeline runs from overlapping with subsequent scheduled executions.

```python
with DAG(
    dag_id="ecom_full_pipeline_dag",
    description="Olist E-Commerce: S3 ingestion → Redshift → dbt transforms",
    schedule_interval="0 2 * * *",  # 02:00 UTC = 09:00 VN time
    start_date=datetime(2024, 1, 1),
    catchup=False,
    default_args=DEFAULT_ARGS,
    tags=["ecommerce", "olist", "dbt", "production"],
    doc_md=__doc__,
    max_active_runs=1,
) as dag:
```

The retry policy for failed tasks is defined centrally through `DEFAULT_ARGS` and is applied consistently across all seven tasks. Each task is retried up to **two times**, with an exponentially increasing retry delay starting at five minutes and capped at thirty minutes.

```python
DEFAULT_ARGS = {
    "owner": "data-engineering",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(minutes=30),
}
```

The seven tasks executed by the DAG are summarized below in their execution order:

| No. | Task ID | Operator Type | Purpose |
| :-- | :--- | :--- | :--- |
| 1 | `ingest_olist_to_s3` | `PythonOperator` | Executes the `ingest()` function in `ingestion/ingest_olist_to_s3.py` to download the Olist dataset from Kaggle and upload it to Amazon S3 |
| 2 | `load_staging_to_redshift` | `PythonOperator` | Executes the `load_all()` function in `ingestion/load_to_redshift.py` to load data from Amazon S3 into the `staging` schema using the Redshift `COPY` command |
| 3 | `dbt_run_staging` | `BashOperator` | Executes `dbt run --select staging` to materialize the seven Staging Models as views |
| 4 | `dbt_test_staging` | `BashOperator` | Executes `dbt test --select staging`, serving as the first data quality checkpoint |
| 5 | `dbt_run_mart` | `BashOperator` | Executes `dbt run --select mart` to materialize the four Data Mart models as tables |
| 6 | `dbt_test_mart` | `BashOperator` | Executes `dbt test --select mart`, serving as the second data quality checkpoint |
| 7 | `notify_success` | `PythonOperator` | Records a success log after all six preceding tasks complete successfully |

The four dbt-related tasks (Tasks 3–6) are all implemented using `BashOperator` and share a common base command:

```python
DBT_DIR = "/usr/app/dbt_project"

# Execute dbt directly (PATH is provided through DBT_ENV below)
DBT_CMD = f"cd {DBT_DIR} && dbt deps && dbt"

DBT_ENV = {
    # Add Airflow local bin directory to PATH
    "PATH": f"/home/airflow/.local/bin:{os.environ.get('PATH', '')}",
    "DBT_PROFILES_DIR": DBT_DIR,
    "DBT_TARGET": "prod",
    **{
        k: os.environ.get(k, "")
        for k in [
            "REDSHIFT_HOST",
            "REDSHIFT_PORT",
            "REDSHIFT_DATABASE",
            "REDSHIFT_USER",
            "REDSHIFT_PASSWORD",
        ]
    },
}
```

The directory `/usr/app/dbt_project` corresponds to the Docker volume mount `./dbt_project:/usr/app/dbt_project` configured in `docker-compose.yml` (Section 5.2). This volume mapping allows the Airflow container to access the same dbt project source code being developed on the host machine without rebuilding the Docker image.

The environment variable `DBT_TARGET="prod"` is explicitly passed to every dbt command, instructing dbt to use the `prod` profile defined in `profiles.yml` (targeting the `analytics` schema with eight execution threads) instead of the `dev` profile. This variable also determines the behavior of the custom schema-naming macro described in Section 5.5.

For example, the `dbt_run_staging` task executes the following command:

```bash
cd /usr/app/dbt_project && dbt deps && dbt run --select staging --target prod
```

Although executing `dbt deps` before every dbt task introduces a small amount of redundant execution time—since packages such as `dbt_utils` and `audit_helper` do not change during a single DAG run—it ensures that every task remains self-contained and independently executable without relying on artifacts generated by previous tasks. This approach aligns with the principle of **idempotent task design**, which is widely recommended for Airflow workflows.

All seven tasks are connected in a single linear dependency chain, with no parallel execution branches:

```python
# Task Dependencies
(
    ingest_task
    >> load_staging_task
    >> dbt_run_staging
    >> dbt_test_staging
    >> dbt_run_mart
    >> dbt_test_mart
    >> notify_task
)
```

The most significant design decision in this sequence is the placement of `dbt_test_staging`. Instead of postponing all validation until the end of the pipeline, Staging-layer tests are executed **before** the Data Mart models are built. This arrangement implements the **fail-fast** principle: if the newly ingested raw data violates critical constraints—for example, duplicate `order_id` values or unexpected `order_status` values—the DAG terminates immediately after Task 4. Consequently, computational resources are not wasted materializing downstream Data Mart models based on data that is already known to be invalid.

Each task is additionally configured with `on_failure_callback=notify_failure`, a Python callback function that records detailed diagnostic information—including the task ID, DAG run ID, retry count, and log location—whenever a task ultimately fails after exhausting all retry attempts. Although the current implementation only writes these details to the Airflow logs, the callback provides a convenient extension point for integrating operational alerting systems such as Slack or email notifications in a production environment.

![Airflow Graph View](image.png)

![Airflow Grid View](image-1.png)
````
````markdown id="5mxt8p"
## 2. Data Quality Testing

The `dbt_test_staging` and `dbt_test_mart` tasks do not perform arbitrary business logic validation. Instead, they execute a set of explicitly defined constraints declared in the `schema.yml` files associated with each model layer. This approach follows dbt's philosophy that **data quality tests should be treated as part of the codebase**, rather than being performed manually after the data has already reached reporting dashboards.

The project employs two categories of tests. The first consists of the built-in generic tests provided by dbt Core: `unique`, `not_null`, and `accepted_values`, which verifies that a column contains only values from a predefined set. The second category uses `dbt_utils.expression_is_true`, a generic test provided by the `dbt_utils` package (declared in `packages.yml`). This test evaluates arbitrary SQL expressions and serves as the primary mechanism for enforcing numerical business constraints, such as ensuring that percentage values remain within the range of 0 to 100.

The table below summarizes the most important data quality constraints defined in `models/staging/schema.yml` and `models/mart/schema.yml`.

| Layer | Model | Column | Test | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| Staging | `stg_orders` | `order_id` | `unique`, `not_null` | Ensures each order appears exactly once |
| Staging | `stg_orders` | `order_status` | `accepted_values` | Restricts order status to one of the eight valid values |
| Staging | `stg_order_items` | `item_price`, `freight_value` | `dbt_utils.expression_is_true: ">= 0"` | Ensures prices and freight charges are never negative |
| Staging | `stg_customers`, `stg_sellers`, `stg_products` | Primary key | `unique`, `not_null` | Ensures uniqueness of dimension tables |
| Staging | `stg_order_reviews` | `review_score` | `accepted_values: [1,2,3,4,5]` | Restricts review scores to the official Olist rating scale |
| Mart | `revenue_daily` | `order_date` | `unique`, `not_null` | Ensures one report row per day |
| Mart | `revenue_daily` | `gmv` | `dbt_utils.expression_is_true: ">= 0"` | Ensures GMV is never negative |
| Mart | `revenue_daily` | `total_orders` | `dbt_utils.expression_is_true: "> 0"` | Ensures each reported day contains at least one order |
| Mart | `seller_performance` | `seller_id` | `unique`, `not_null` | Ensures one performance record per seller |
| Mart | `seller_performance` | `seller_score` | `dbt_utils.expression_is_true: "between 0 and 100"` | Ensures the performance score remains within the designed 0–100 range |
| Mart | `fct_customer_cohorts` | `retention_rate_pct` | `dbt_utils.expression_is_true: "between 0 and 100"` | Ensures retention rates are valid percentages |
| Mart | `fct_customer_cohorts` | `months_since_first_order` | `dbt_utils.expression_is_true: ">= 0"` | Ensures no negative elapsed time since the first purchase |

In addition to tests attached to the Staging Models, the raw source tables (`raw_orders`, `raw_customers`, `raw_sellers`, `raw_products`, `raw_order_payments`, and `raw_order_reviews`) are also associated with similar tests (`unique`, `not_null`, and `accepted_values`) directly within the source definitions in `sources.yml`. Consequently, several critical data quality constraints are validated **as early as possible**, before the data passes through any Staging Model transformations.

The `sources.yml` file also defines **source freshness** thresholds, issuing a warning if a raw table has not been refreshed within **25 hours** and raising an error if more than **49 hours** have elapsed since the latest data load. However, the corresponding command (`dbt source freshness`) has not yet been incorporated as a dedicated Airflow task within the DAG. Instead, it remains an available validation tool that can be executed manually. Integrating this command into the orchestration workflow represents a potential enhancement for achieving a more comprehensive automated data quality framework.

When a `dbt test` task is executed, dbt produces a summary report containing the number of tests in four possible states—`PASS`, `WARN`, `ERROR`, and `SKIP`—together with the total number of executed tests. A typical output follows the format:

```text
Done. PASS=12 WARN=0 ERROR=0 SKIP=0 TOTAL=12
```

Any test that returns an `ERROR` causes the dbt process to exit with a non-zero exit code. The corresponding `BashOperator` within Airflow marks the task as failed and triggers the retry policy configured in `DEFAULT_ARGS`. If the task still fails after all retry attempts have been exhausted, all downstream tasks—including `notify_success`—are skipped.

One important technical consideration concerns the interpretation of the second validation checkpoint (`dbt_test_mart`). Because `dbt_run_mart` executes **before** the tests and materializes the Data Mart models as physical tables (`materialized='table'`) using `CREATE TABLE AS` (or table replacement), the newly generated data has already been written into the `mart` schema by the time `dbt_test_mart` begins execution. Consequently, if `dbt_test_mart` fails, the DAG terminates and alerts the operations team, but it **does not automatically roll back** or remove the invalid data that has already been materialized.

Within the scope of this project, this behavior is considered acceptable because the pipeline runs only once per day and failed executions are monitored through Airflow logs. However, in a production environment with multiple downstream consumers querying the Data Mart directly, a more robust deployment strategy—such as the **Write–Audit–Publish** pattern (writing to temporary tables, validating them, and only then swapping them into production)—would be preferable to ensure that unvalidated data is never exposed to end users.

![Log Output of the `dbt_test_mart` Task](5.4.3-dbt-test-pass.png)
````
