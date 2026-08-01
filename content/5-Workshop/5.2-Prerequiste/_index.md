---
title: "5.2 - Environment Preparation"
date: 2026-07-21
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Before deploying the core components of the system, the development environment was fully prepared, including the hardware platform, AWS cloud resources, containerized runtime environment, and development toolchain. This section describes the complete preparation process, together with practical considerations encountered during the implementation.

## 1. Hardware Requirements

Because the system runs multiple Docker containers simultaneously—including the Airflow Webserver, Airflow Scheduler, and Airflow's internal PostgreSQL database—while maintaining continuous connectivity to AWS cloud services, the workstation specifications were determined based on Airflow's minimum requirements with additional capacity reserved for concurrent dbt workloads.

The recommended hardware specifications are shown below:

| Component | Minimum Specification | Recommended Specification (Used in This Project) |
| :--- | :--- | :--- |
| **Operating System** | Windows 10/11 Pro or Enterprise (WSL2), macOS 12+, Ubuntu 20.04 LTS | Windows 11 Pro (WSL2), macOS Apple Silicon (M1/M2/M3), Ubuntu 22.04 LTS |
| **CPU** | 4 Cores / 8 Threads | 8 Cores / 16 Threads |
| **Memory** | 8 GB RAM | 16 GB RAM or higher (Airflow and dbt consume significant memory when multiple models execute concurrently) |
| **Available Storage** | 20 GB SSD | 50 GB NVMe SSD (to accommodate locally downloaded Olist data and Docker images) |

During the implementation, 8 GB of RAM proved insufficient when the Airflow Scheduler, Webserver, and dbt processes were running simultaneously. Therefore, **16 GB of RAM is strongly recommended** rather than being considered an optional enhancement.

---

## 2. AWS Account and IAM Configuration

The project primarily relies on **Amazon S3** and **Amazon Redshift Serverless**, while **AWS Glue** is used for data cataloging purposes. AWS access control is organized into two distinct layers:

1. **Operator permissions**, used for provisioning and managing cloud resources.
2. **Service execution permissions**, allowing AWS services to interact with each other automatically during pipeline execution.

Separating these two permission models follows AWS security best practices by preventing automated services from receiving unnecessarily broad privileges.

### 2.1. AWS Account and IAM User

- An active AWS account with billing enabled and access to the **AWS Management Console**.
- A dedicated IAM user (for example, `data-pipeline-admin`) created specifically for deployment activities instead of using the root account, following AWS security recommendations.
- The IAM user is assigned two managed policies:
  - `AmazonS3FullAccess`
  - `AmazonRedshiftFullAccess`

  These permissions are appropriate for a development environment. In production, they should be replaced with a least-privilege policy that grants access only to the required buckets and Redshift resources.

- A pair of **Access Key ID** and **Secret Access Key** is generated for programmatic authentication through the AWS CLI and Python scripts.

### 2.2. IAM Role for Amazon Redshift

Unlike the IAM user used by developers, the project defines a dedicated **IAM Role** that enables Amazon Redshift Serverless to read data directly from Amazon S3 when executing the `COPY` command.

This role is provisioned through Terraform (`infrastructure/terraform/main.tf`) rather than manually configured in the AWS Console, ensuring reproducibility across environments.

The attached IAM policy follows the **principle of least privilege**, allowing only:

- `s3:GetObject`
- `s3:ListBucket`

and only for the project's **Raw** and **Processed** S3 buckets.

This is intentionally much more restrictive than the `AmazonS3FullAccess` policy granted to the deployment IAM user.

### 2.3. Initial AWS Infrastructure

Before beginning the deployment process described in Section 5.3, the following AWS resources are prepared:

1. **Amazon S3 Buckets**

   Two buckets are created following the project's naming convention:

   - `ecom-pipeline-raw-dev`
   - `ecom-pipeline-processed-dev`

   Both buckets are deployed in the **ap-southeast-2** region to minimize latency and avoid inter-region transfer costs.

   The **Block all public access** setting remains enabled for both buckets.

2. **Amazon Redshift Serverless**

   Instead of provisioning a traditional Redshift cluster with fixed compute nodes, the project uses **Amazon Redshift Serverless**, consisting of:

   - A **Namespace**, which stores database objects and security configurations.
   - A **Workgroup**, which defines the compute capacity using **Redshift Processing Units (RPUs)**.

   The serverless deployment model automatically scales based on workload demand, making it well suited for the moderate size of the Olist dataset.

![Amazon S3 Bucket](image-1.png)

---

## 3. Containerized Environment (Docker)

The entire platform is containerized using Docker to ensure a consistent runtime environment across development machines and deployment environments.

### 3.1. Installing Docker Engine and Docker Compose

The project uses:

- **Docker Desktop** on Windows and macOS.
- **Docker Engine** together with the `docker-compose-plugin` on Linux.

Docker Compose version **2.20.0** or later is required.

### 3.2. Services Defined in `docker-compose.yml`

The project's `docker-compose.yml` defines the following services using the `apache/airflow:2.8.1-python3.11` image as the common base:

1. **postgres**

   PostgreSQL 15 serving exclusively as Airflow's metadata database, storing DAG execution history and task states. It does **not** contain business data.

2. **airflow-webserver**

   Hosts the Airflow Web UI on port **8080** and performs health checks every 30 seconds.

3. **airflow-scheduler**

   Continuously scans the DAG directory, schedules workflows, and dispatches tasks for execution.

4. **airflow-init**

   A one-time initialization container responsible for:

   - Migrating the Airflow metadata database.
   - Creating the default administrator account.

5. **dbt**

   A standalone service placed under the **manual** Docker Compose profile. It is intended for manually executing dbt commands without relying on Airflow.

AWS credentials

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_DEFAULT_REGION`

and Redshift connection settings

- `REDSHIFT_HOST`
- `REDSHIFT_PORT`
- `REDSHIFT_DATABASE`
- `REDSHIFT_USER`
- `REDSHIFT_PASSWORD`

are injected into containers through a `.env` file rather than being hardcoded.

### Security Consideration

During the project review, it was observed that the Kaggle authentication variables (`KAGGLE_USERNAME` and `KAGGLE_KEY`) were hardcoded directly in `docker-compose.yml` instead of being referenced from the `.env` file.

This configuration should be corrected before making the repository publicly available (for example, on GitHub), because credentials exposed in Git history cannot be fully removed simply by deleting the file.

The recommended approach is to:

- Move both credentials into the `.env` file (already listed in `.gitignore`), or
- Store them securely using **AWS Secrets Manager** or another secret management solution.

Any exposed Kaggle API key should also be revoked and regenerated.

---

## 4. Development Tools

### 4.1. Local Python Environment

Python **3.11** or later is installed locally for developing scripts in the `ingestion/` directory and for running dbt CLI commands outside Docker when needed.

The project's `requirements.txt` includes the primary libraries:

- `boto3`
- `pandas`
- `redshift-connector`
- `dbt-core`
- `dbt-redshift`
- `kaggle`

The Redshift adapter is installed using:

```bash
pip install dbt-core==1.7.3 dbt-redshift==1.7.1
```

### 4.2. Visual Studio Code Extensions

Visual Studio Code is used as the primary IDE throughout the project, together with the following extensions:

- **Python (`ms-python.python`)** – IntelliSense, linting, and debugging support for the Python ingestion scripts.
- **dbt Power User (`innoverio.vscode-dbt-power-user`)** – SQL preview from compiled Jinja templates, lineage graph visualization, and one-click model execution.
- **Docker (`ms-azuretools.vscode-docker`)** – Container management and log monitoring directly from the IDE.
- **YAML (`redhat.vscode-yaml`)** – Syntax validation for YAML configuration files such as `docker-compose.yml`, `profiles.yml`, `dbt_project.yml`, and `schema.yml`.

---

## 5. Pre-deployment Readiness Checklist

Before proceeding to the detailed deployment process in Section 5.3, all of the following requirements have been verified. This checklist serves as evidence that the development environment is fully prepared before provisioning AWS resources.

| No. | Verification Item | Verification Method | Status |
| :-- | :--- | :--- | :--- |
| 1 | Docker Engine and Docker Compose are installed correctly | `docker --version` and `docker compose version` | Passed |
| 2 | All project containers start successfully | `docker ps` shows all Airflow and PostgreSQL containers running | Passed |
| 3 | IAM User successfully authenticates through AWS CLI | `aws sts get-caller-identity` returns the expected IAM user | Passed |
| 4 | Amazon S3 bucket is accessible | `aws s3 ls s3://<bucket-name>` executes without errors | Passed |
| 5 | dbt CLI is installed correctly | `dbt --version` displays both `dbt-core` and `dbt-redshift` versions | Passed |
| 6 | Airflow Web UI is accessible | Successfully log in at `http://localhost:8080` | Passed |

![`docker ps` output](5.2-docker-ps.png)

![IAM Role](image.png)
```