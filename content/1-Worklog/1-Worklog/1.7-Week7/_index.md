---
title: "Week 7 Worklog"
date: 2026-07-13
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Week 7 Objectives:

* Codify all AWS resources used so far (S3, Redshift Serverless, IAM, Glue) as Terraform, so the whole stack becomes reproducible from scratch.
* Reconcile the manually-provisioned resources from Weeks 2–4 with a clean Terraform state, without losing existing data.
* Package the local development environment (Airflow + Postgres + dbt) with Docker Compose.
* Apply security and cost-control best practices at the infrastructure level (least privilege, public access blocks, S3 lifecycle rules).

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                                   | Start Date | Completion Date | Reference Material                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | ----------------------------------------- |
| 2   | - Learn Terraform basics: providers, resources, variables, state, `plan`/`apply` <br> - Import the manually-created S3 bucket and Redshift Serverless namespace into Terraform state (`terraform import`) so future changes go through code, not console clicks | 07/13/2026 | 07/13/2026      | Terraform AWS Provider docs |
| 3   | - Write `aws_s3_bucket` + `aws_s3_bucket_lifecycle_configuration` (90-day transition to INTELLIGENT_TIERING, 365-day expiration) + `aws_s3_bucket_public_access_block`                                 | 07/14/2026 | 07/14/2026      | |
| 4   | - Write `aws_iam_role`/policy for Redshift-to-S3 access <br> - Write `aws_redshiftserverless_namespace`/`workgroup` (8 base RPU, `publicly_accessible = false`) + security group restricted to port 5439 | 07/15/2026 | 07/15/2026      | |
| 5   | - Write `aws_glue_catalog_database` + crawler <br> - Set up `variables.tf` so the same Terraform code deploys both `dev` and `prod` by changing one `environment` variable                              | 07/16/2026 | 07/16/2026      | |
| 6   | - Write `docker-compose.yml` with `x-airflow-common` YAML anchor (postgres, airflow-webserver, airflow-scheduler, airflow-init, dbt service) <br> - **Test:** `terraform destroy` + `terraform apply` from scratch in a fresh AWS account/region, confirm the whole stack rebuilds identically | 07/17/2026 | 07/18/2026 | |


### Week 7 Achievements:

* Brought all AWS infrastructure under Terraform: S3 (raw bucket + lifecycle + public access block), Redshift Serverless (namespace + workgroup + security group), IAM role for Redshift→S3 access, and Glue catalog + crawler — 4 resource groups total.

* Reconciled the resources created manually in Weeks 2–4 into Terraform state via `terraform import`, so the project didn't need to tear down and lose the data already loaded — an important lesson: infrastructure-as-code doesn't have to mean starting over, it can also mean formalizing what already works.

* Verified reproducibility directly: ran `terraform destroy` then `terraform apply` in a separate AWS region, and confirmed the pipeline could be pointed at the newly created bucket/workgroup with no manual console steps.

* Implemented cost controls at the infrastructure level: S3 lifecycle rule moves data older than 90 days to `INTELLIGENT_TIERING` and expires it after 365 days; Redshift Serverless capped at 8 base RPU rather than a fixed-size provisioned cluster.

* Implemented security controls: `aws_s3_bucket_public_access_block` blocks all public access to the raw bucket; Redshift Serverless workgroup set to `publicly_accessible = false`; security group only opens port 5439 to internal traffic.

* Completed `docker-compose.yml` for the local dev environment, using the `x-airflow-common` YAML anchor to avoid repeating environment/volume config three times across webserver, scheduler, and init containers.

* Learned a practical Terraform lesson: since S3 buckets and Redshift Serverless resources had already existed since Week 2–4, the natural next step after "learn Terraform" wasn't provisioning from a blank slate — it was importing and formalizing existing infrastructure, which is closer to how IaC adoption actually happens on a running system.
