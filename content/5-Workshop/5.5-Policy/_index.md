---
title: "5.5 - Governance & Access Configuration"
date: 2026-07-27
weight: 5
chapter: false
pre: " <b> 5.5 </b> "
---

To ensure the project architecture remains scalable, maintainable, and easy to manage, configure the model hierarchy and governance rules in the `dbt_project.yml` file.

### Standardized `dbt_project.yml` Configuration

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

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  ecom_pipeline:
    staging:
      +materialized: view
      +schema: staging
      +tags: ["staging"]

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
```