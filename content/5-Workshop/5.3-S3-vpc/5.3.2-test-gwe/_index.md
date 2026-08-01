---
title: "Verifying Connectivity and Configuring the dbt Profile"
date: 2026-07-22
weight: 2
chapter: false
pre: " <b> 5.3.2 </b> "
---

After the storage infrastructure, data warehouse, and private connectivity through the Gateway VPC Endpoint have been configured in Section 5.3.1, the next step is to verify that the entire connection path functions as intended before building the dbt models in Section 5.4.

Verification is performed indirectly through **dbt**. If dbt successfully connects to Amazon Redshift and a `COPY` command from Amazon S3 executes without any network-related errors, it can be concluded that both the Redshift connection and the Gateway VPC Endpoint routing are operating correctly.

## 1. Configuring `profiles.yml`

dbt uses the `profiles.yml` file to store data warehouse connection settings, keeping connection credentials completely separate from model source code. This prevents sensitive information from being accidentally committed to version control.

The project's configuration defines two deployment targets—`dev` and `prod`—under a shared profile named `ecom_pipeline`:

```yaml
ecom_pipeline:
  target: "{{ env_var('DBT_TARGET', 'dev') }}"
  outputs:
    dev:
      type: redshift
      host: "{{ env_var('REDSHIFT_HOST') }}"
      port: "{{ env_var('REDSHIFT_PORT', '5439') | int }}"
      user: "{{ env_var('REDSHIFT_USER') }}"
      password: "{{ env_var('REDSHIFT_PASSWORD') }}"
      dbname: "{{ env_var('REDSHIFT_DATABASE', 'dev') }}"
      schema: analytics_dev
      threads: 4
      ra3_node: true
      connect_timeout: 30

    prod:
      type: redshift
      host: "{{ env_var('REDSHIFT_HOST') }}"
      port: "{{ env_var('REDSHIFT_PORT', '5439') | int }}"
      user: "{{ env_var('REDSHIFT_USER') }}"
      password: "{{ env_var('REDSHIFT_PASSWORD') }}"
      dbname: "{{ env_var('REDSHIFT_DATABASE', 'dev') }}"
      schema: analytics
      threads: 8
      ra3_node: true
      connect_timeout: 30
```

All sensitive connection parameters—including the hostname, username, and password—are loaded from environment variables using dbt's `env_var()` function instead of being hardcoded into the configuration file. This approach is consistent with the environment variable management strategy described in Section 5.2.

The parameter `ra3_node: true` is enabled because the project's Redshift Serverless Workgroup is based on the RA3 architecture, allowing cross-database queries when required.

Different thread counts are configured for the two environments:

- **Development (`dev`)** uses **4 threads** to reduce Redshift Processing Unit (RPU) consumption during development.
- **Production (`prod`)** uses **8 threads** to maximize execution performance.

Another notable difference is the target schema:

- `analytics_dev` for the development environment.
- `analytics` for the production environment.

The mechanism behind this schema selection is explained in detail in the `generate_schema_name.sql` macro discussed in Section 5.5.

---

## 2. Testing the Connection with `dbt debug`

After saving the configuration, the connection is verified by executing the following command from the `dbt_project/` directory:

```bash
dbt debug
```

The command performs several validation steps in sequence:

1. Validates the syntax of `profiles.yml`.
2. Resolves all required environment variables.
3. Establishes a TCP connection to Amazon Redshift on port **5439**.
4. Executes a simple test query.

If all infrastructure components described in Section 5.3.1 have been configured correctly, dbt reports that all verification checks—including **Connection**, **Profile**, **Debug**, and **All checks**—have successfully passed.

If `dbt debug` reports a connection timeout, the most common causes are not related to the dbt configuration itself but rather to the underlying network configuration. Typical issues include:

- The Redshift Security Group does not allow inbound traffic from the machine running dbt.
- The machine executing dbt is located outside the VPC and does not have network access through a VPN or Bastion Host to the private subnet hosting the Redshift Workgroup.

For this reason, the project's Docker Compose environment executes dbt from within the same container network used by Apache Airflow rather than connecting directly from a developer's workstation.

---

## 3. Verifying the Gateway VPC Endpoint Using a `COPY` Command

A successful `dbt debug` only verifies the **control plane** connection between dbt and Amazon Redshift. It does not confirm that the **data plane** between Redshift and Amazon S3 is functioning correctly through the Gateway VPC Endpoint.

To verify the complete data path, the following test `COPY` command is executed directly in **Redshift Query Editor v2** to load a sample file from the `ecom-pipeline-raw-prod` bucket into a temporary table within the `staging` schema:

```sql
COPY staging.raw_customers_test
FROM 's3://ecom-pipeline-raw-prod/olist/year=2024/month=01/day=15/olist_customers_dataset.csv'
IAM_ROLE 'arn:aws:iam::<account-id>:role/ecom-pipeline-redshift-s3-prod'
REGION 'ap-southeast-2'
FORMAT AS CSV
IGNOREHEADER 1;
```

Successful execution of this command, together with the reported number of imported rows, confirms that:

1. The IAM Role attached to the Redshift Namespace has sufficient permission to read objects from the S3 bucket.
2. Network communication between Amazon Redshift and Amazon S3 is functioning correctly through the Gateway VPC Endpoint, indicating that both the endpoint and the associated Route Table configuration described in Section 5.3.1 have been implemented successfully.

After verification, the temporary table `raw_customers_test` is removed using a `DROP TABLE` statement, as this step serves only as an infrastructure validation. The complete data loading process into the production `staging` tables is presented later in Section 5.4.1.