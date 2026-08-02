---
title: "Governance Configuration & Access Control"
date: 2026-07-27
weight: 5
chapter: false
pre: " <b> 5.5 </b> "
---

This section presents the governance mechanisms implemented throughout the project, organized into two layers. The first focuses on governance within the data transformation layer—how dbt is centrally configured so that every model follows the same conventions for schema assignment, materialization, and naming. The second covers infrastructure and development governance—controlling which users or services are allowed to access AWS resources, and how source code is automatically validated before being merged into the main branch. Finally, this section concludes an open topic introduced in Section 5.3.1 regarding the temporary **Full Access** policy assigned to the Gateway VPC Endpoint during the initial infrastructure setup.

## 1. Centralized Governance Configuration with `dbt_project.yml`

Instead of repeatedly specifying `materialized`, `schema`, and `tags` inside every model using `{{ config(...) }}`, most of these properties are defined once in `dbt_project.yml` according to the project directory hierarchy. Every model within a given directory automatically inherits the corresponding configuration.

```yaml
name: 'ecom_pipeline'
version: '1.0.0'
config-version: 2

profile: 'ecom_pipeline'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

#
# Model configs per layer
#
models:
  ecom_pipeline:
    staging:
      +materialized: view
      +schema: staging
      +tags: ["staging"]

    intermediate:
      +materialized: ephemeral
      +tags: ["intermediate"]

    mart:
      +materialized: table
      +schema: mart
      +tags: ["mart"]

      finance:
        +materialized: table
        revenue_daily:
          +sort: ["order_date"]
          +dist: "order_date"

      customers:
        +materialized: table
        fct_customer_cohorts:
          +sort: ["cohort_month"]

      operations:
        +materialized: table

vars:
  # Used to filter data in development so dbt runs fast locally
  start_date: '2016-01-01'
  # Override in prod: dbt run --vars '{"start_date": "2016-01-01"}'
```

The `models.ecom_pipeline` section clearly reflects the three-layer architecture described throughout Section 5.4. Models under `staging` are materialized as **views**, stored in the `staging` schema, and tagged with `staging`. Models under `mart` are materialized as **tables**, stored in the `mart` schema, and tagged with `mart`. The three subdirectories—`finance`, `customers`, and `operations`—correspond directly to the three business domains introduced in Section 5.4.2. Additional Redshift-specific configurations, such as `sort` and `dist` keys, are also defined centrally for models like `revenue_daily` and `fct_customer_cohorts`. Some models, such as `seller_performance`, override the default configuration by specifying `dist` directly inside their own `{{ config(...) }}` block, since dbt allows file-level configurations to take precedence over directory-level defaults whenever model-specific customization is required.

Another notable configuration is the predefined `intermediate` layer, materialized as **ephemeral**. Ephemeral models are inlined into downstream SQL queries as Common Table Expressions (CTEs) and therefore do not create physical tables or views inside Redshift. Although the project currently contains no models under `models/intermediate/`, this layer has been intentionally reserved for future development, when transformation logic becomes sufficiently complex to warrant reuse across multiple Data Mart models without introducing additional physical database objects.

The global variable `vars.start_date` is referenced by all four Data Mart models (`revenue_daily`, `category_revenue_monthly`, `fct_customer_cohorts`, and `seller_performance`) through `{{ var("start_date") }}` to restrict the range of processed data. Its default value, `2016-01-01`, corresponds to the earliest timestamp available in the Olist dataset, meaning that no data is excluded in the production environment. The variable becomes particularly useful during local development, where developers can shorten execution time by overriding it at runtime, for example:

```bash
dbt run --vars '{"start_date": "2024-01-01"}'
```

without modifying the SQL logic of any model.

---

## 2. Environment-Specific Schema Naming Convention

One of the most common risks in collaborative dbt projects is that a developer accidentally executes `dbt run` against the production schemas while testing changes locally, thereby overwriting data used by scheduled production pipelines. This project avoids that risk through a custom macro, `macros/generate_schema_name.sql`, which overrides dbt's default schema naming behavior.

```sql
-- macros/generate_schema_name.sql
--
-- Custom schema naming convention:
--   dev  target: analytics_dev__staging, analytics_dev__mart
--   prod target: staging, mart  (no prefix)

{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}

    {%- if target.name == 'prod' -%}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ custom_schema_name | trim }}
        {%- endif -%}

    {%- else -%}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ default_schema }}_{{ custom_schema_name | trim }}
        {%- endif -%}
    {%- endif -%}

{%- endmacro %}
```

Combined with the two targets defined in `profiles.yml` (`dev` using the default schema `analytics_dev`, and `prod` using `analytics`), this macro produces different physical schema names depending on the execution target:

| Execution Target (`--target`) | `custom_schema_name` | Generated Schema |
| :--- | :--- | :--- |
| `prod` | `staging` | `staging` |
| `prod` | `mart` | `mart` |
| `dev` | `staging` | `analytics_dev_staging` |
| `dev` | `mart` | `analytics_dev_mart` |

One small inconsistency is worth noting. The comment at the beginning of the macro uses examples such as `analytics_dev__staging` (double underscore), whereas the actual string concatenation expression (`{{ default_schema }}_{{ custom_schema_name }}`) generates only a single underscore (`analytics_dev_staging`), matching the examples shown in the table above. This discrepancy exists only in the documentation comment and does not affect the macro's functionality, although updating the comment would improve clarity for future maintainers.

When executed with `--target prod`—the configuration explicitly passed by Airflow through `DBT_TARGET="prod"` in Section 5.4.3—the macro ignores the default schema prefix and returns exactly the schema names defined in `dbt_project.yml` (`staging` and `mart`), matching the schemas created manually during the infrastructure setup in Section 5.3.1.

Conversely, when a developer executes `dbt run` locally without explicitly specifying `--target prod`, dbt defaults to the `dev` target as configured in `profiles.yml` (`target: "{{ env_var('DBT_TARGET', 'dev') }}"`). In that case, all models are written into schemas prefixed with `analytics_dev_`, ensuring complete isolation from the production environment. This mechanism allows developers to experiment freely without risking accidental modification of production datasets, while eliminating the need to manually change schema names during development.

---

## 3. Model Naming Conventions and Package Dependency Management

In addition to centralized schema configuration, the project follows several naming conventions that improve maintainability as the number of models grows.

| Convention | Applies To | Examples |
| :--- | :--- | :--- |
| Prefix `stg_` | All staging models | `stg_orders`, `stg_sellers` |
| Prefix `fct_` | Fact tables in the Mart layer | `fct_customer_cohorts` |
| Descriptive aggregate names | Aggregate Mart models | `revenue_daily`, `seller_performance` |
| One subdirectory per business domain | Mart layer | `mart/finance/`, `mart/customers/`, `mart/operations/` |
| Tags by layer and business domain | All models | `tags: ['mart', 'finance', 'daily']` |

The tagging convention serves not only as documentation but also as an operational mechanism. The Airflow tasks `dbt_run_staging` and `dbt_run_mart`, described in Section 5.4.3, execute `dbt run --select staging` and `dbt run --select mart`, respectively. These selectors rely on dbt's directory-based selection mechanism (which aligns with the assigned tags), allowing Airflow to execute each layer independently without explicitly listing every model.

External dependencies are managed through `packages.yml`, which specifies acceptable version ranges instead of pinning exact versions.

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
  - package: dbt-labs/audit_helper
    version: [">=0.10.0", "<1.0.0"]
```

When `dbt deps` is executed, dbt resolves concrete package versions within the specified ranges and records them in `package-lock.yml` (`dbt_utils` resolves to version `1.4.1`, while `audit_helper` resolves to `0.14.0`). This lock file plays the same role as dependency lock files in other package management ecosystems, ensuring that every environment—including developers' local machines, CI containers, and the production Airflow environment—installs exactly the same package versions that have already been validated. Consequently, updates to external packages cannot inadvertently alter the behavior of tests such as `dbt_utils.expression_is_true`, which are extensively used throughout the project's data quality validation workflow described in Section 5.4.3.
## 4. Infrastructure Access Control: From IAM Roles to the VPC Endpoint Policy Gap

As described in Section 5.2, Redshift's access to Amazon S3 is managed through a dedicated IAM Role, separate from operator permissions, with a policy restricted to exactly two S3 actions on only the two buckets used by the project:

```hcl
resource "aws_iam_policy" "redshift_s3_policy" {
  name = "${local.project}-redshift-s3-policy-${local.environment}"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:ListBucket"]
        Resource = [
          aws_s3_bucket.raw.arn,
          "${aws_s3_bucket.raw.arn}/*",
          aws_s3_bucket.processed.arn,
          "${aws_s3_bucket.processed.arn}/*",
        ]
      }
    ]
  })
}
```

This represents the IAM layer of access control, which determines **which AWS service is allowed to invoke which S3 APIs**. However, Section 5.3.1 also introduced another, independent layer of control: the policy attached directly to the Gateway VPC Endpoint, which determines **which S3 requests are allowed to traverse the private network path**. That section also noted that the temporary **Full Access** endpoint policy used during the initial infrastructure setup would later be tightened in this section.

A review of the current Terraform codebase (`infrastructure/terraform/main.tf`) shows that the project **does not yet define** any `aws_vpc_endpoint` resource. In other words, the Gateway VPC Endpoint described in Section 5.3.1 currently exists only as a manually configured resource in the AWS Console and has not yet been incorporated into the Infrastructure as Code (IaC) repository. This represents a governance gap that should be addressed so the Terraform configuration fully reflects the deployed infrastructure and fulfills the design commitment stated in Section 5.3.1. The recommended Terraform configuration is shown below:

```hcl
resource "aws_vpc_endpoint" "s3_gateway" {
  vpc_id            = var.vpc_id
  service_name      = "com.amazonaws.${var.aws_region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = var.private_route_table_ids

  # Restrict access from Full Access to only the two project buckets,
  # instead of allowing access to every S3 bucket in the AWS account.
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [
          aws_s3_bucket.raw.arn,
          "${aws_s3_bucket.raw.arn}/*",
          aws_s3_bucket.processed.arn,
          "${aws_s3_bucket.processed.arn}/*",
        ]
      }
    ]
  })

  tags = local.tags
}
```

Compared with the original **Full Access** endpoint policy, the key difference lies in the `Resource` block. Instead of implicitly allowing requests to every S3 bucket in the AWS account, the revised policy explicitly lists only the ARNs of the project's two buckets (`raw` and `processed`).

From a security perspective, this restriction provides an additional **defense-in-depth** layer beyond the existing IAM Role. Even if another resource within the same VPC—for example, an EC2 instance unrelated to the data platform—were mistakenly granted the `AmazonS3FullAccess` managed policy, the Gateway VPC Endpoint would still permit traffic only to the two authorized buckets, preventing access to any other S3 buckets in the account.

---

## 5. Governance Through CI/CD (GitHub Actions)

In addition to the governance mechanisms described above, the project uses GitHub Actions (`.github/workflows/ci.yml`) as an automated quality gate before code changes are merged into the `main` branch. The workflow is triggered on every push to the `main` and `develop` branches, as well as on every Pull Request targeting `main`. Three jobs are executed on every run:

| Job | Purpose |
| :--- | :--- |
| `lint-python` | Verifies Python formatting with `black --check`, import ordering with `isort --check-only`, and code quality with `flake8` for the `ingestion/` and `airflow/` directories. |
| `dbt-compile` | Installs `dbt-core==1.7.3` and `dbt-redshift==1.7.1`, runs `dbt deps`, and executes `dbt compile --target dev` using a dummy Redshift connection (`dummy.host`). Compilation is intended only to detect SQL and Jinja syntax errors and therefore does not require an actual Redshift connection. |
| `terraform-validate` | Executes `terraform init -backend=false`, `terraform validate`, and `terraform fmt -check` within `infrastructure/terraform` to ensure the infrastructure code remains syntactically valid and consistently formatted. |

Using dummy connection information (`REDSHIFT_HOST: "dummy.host"` and `REDSHIFT_USER: "ci_user"`) in the `dbt-compile` job is a deliberate technical decision. Since the objective is only to validate SQL and Jinja syntax and confirm that all `ref()` and `source()` references resolve correctly, there is no need to establish a network connection to the production Amazon Redshift Serverless environment. This approach keeps the CI pipeline lightweight, fast, and independent of production credentials when validating Pull Requests.

The workflow also includes a fourth deployment job that is currently commented out, intended to automatically execute `dbt run --target prod` whenever changes are merged into the `main` branch:

```yaml
# Deploy to prod (on main push only)
# Uncomment and configure when ready for automated deploys
# deploy:
#   name: Deploy dbt to Production
#   needs: [lint-python, dbt-compile]
#   if: github.ref == 'refs/heads/main'
#   runs-on: ubuntu-latest
#   environment: production
```

Keeping this deployment job disabled is an appropriate and cautious decision at the current stage of the project. Automatically executing `dbt run --target prod` immediately after every merge into `main`, without a manual approval step or an intermediate staging environment, could allow insufficiently validated model changes to affect production data directly. This deployment workflow is better introduced in a future phase, after adding an approval mechanism—such as GitHub Environments with **required reviewers**—for the already-defined `production` environment.

---

## 6. Notes on Consistency Between Documentation and Source Code

Keeping the project documentation aligned with the actual implementation is itself an important governance practice, although it is often overlooked compared with technical control mechanisms.

During the review conducted for Sections 5.4 and 5.5, two inconsistencies were identified between `README.md` and the current state of the repository that should be addressed in a future documentation update.

First, the **Project Structure** section lists a `scripts/` directory as part of the repository layout, although this directory does not exist in the current codebase.

Second, the **Skills Demonstrated** section claims that the project implements **incremental dbt models for large tables**, whereas all four Data Mart models described in Section 5.4.2 are currently configured with `materialized='table'`, meaning they perform a full refresh on every execution. No model currently uses dbt's `incremental` materialization strategy.

Both items represent planned future enhancements rather than completed functionality. Updating the documentation to reflect the project's actual implementation would improve its accuracy and ensure that the documented architecture remains consistent with the deployed codebase.