---
title: "Week 3 Worklog"
date: 2026-06-15
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Week 3 Objectives:

* Build the Bronze layer: a Python script that downloads the Olist dataset and uploads it to S3, safely and repeatably.
* Implement idempotency: running the script twice for the same day must not duplicate data or waste time.
* Implement data integrity checks (MD5 checksum) and a manifest file for observability.
* Set up the project's Python environment and repository structure.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                                   | Start Date | Completion Date | Reference Material                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | ----------------------------------------- |
| 2   | - Set up project repo structure (`ingestion/`, `dbt_project/`, `airflow/`, `infrastructure/`) <br> - Set up Python 3.11 virtualenv, `requirements.txt` (boto3, pandas, kaggle)                          | 06/15/2026 | 06/15/2026      | |
| 3   | - Configure Kaggle API credentials <br> - Write `download_olist_dataset()`: pull the 9 CSVs from Kaggle into a local temp directory                                                                    | 06/16/2026 | 06/16/2026      | Kaggle API docs |
| 4   | - Design the S3 date-partitioning scheme: `s3://{bucket}/olist/year=YYYY/month=MM/day=DD/` <br> - Write `check_already_ingested()` for idempotency: skip upload if the partition already exists (unless `--force`) | 06/17/2026 | 06/17/2026      | |
| 5   | - Write `upload_to_s3()`: attach MD5 checksum, file size, and ingestion timestamp as S3 object metadata <br> - Write `validate_csv()`: basic sanity checks (row count, null check) before upload         | 06/18/2026 | 06/18/2026      | |
| 6   | - Assemble the main `ingest()` function end-to-end <br> - Write `_manifest.json` after each run (per-file success/failure log) <br> - **Test:** run twice for the same day, confirm the second run is skipped | 06/19/2026 | 06/20/2026 | |


### Week 3 Achievements:

* Repository scaffolded with a clear separation of concerns: `ingestion/`, `dbt_project/`, `airflow/dags/`, `infrastructure/terraform/`, matching the architecture designed in Week 2.

* Completed `ingestion/ingest_olist_to_s3.py`, with the core `ingest(run_date, force=False)` function that:
  1. checks whether the day's partition already exists on S3 (idempotency),
  2. downloads the 9 Olist CSVs from Kaggle,
  3. validates each file,
  4. uploads to S3 with MD5/size/timestamp metadata,
  5. writes a `_manifest.json` summarizing the run.

* Confirmed idempotency works as intended: re-running `ingest()` for a day already ingested logs `"already exists, skipping"` and returns immediately, instead of re-downloading ~50MB of CSVs and re-uploading to S3.

* Understood why MD5 metadata matters: it gives a way to verify file integrity later without re-fetching the source, which becomes important once the pipeline runs unattended on a schedule.

* First successful end-to-end manual run: all 9 files landed under the correct date partition in S3, with a valid manifest confirming 9/9 uploads succeeded.

* Learned a practical lesson about idempotency design: checking "does the partition prefix exist" is cheap (one S3 `list_objects` call) compared to checking file-by-file, which matters once this step runs daily inside Airflow.
