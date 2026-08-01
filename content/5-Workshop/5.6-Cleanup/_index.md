---
title: "Resource Cleanup"
date: 2026-07-28
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

To avoid unexpected AWS charges after completing this workshop, clean up your cloud resources by following the steps below.

### Step 1: Disable the Airflow DAG

Open the Airflow Web UI and switch the `ecom_full_pipeline_dag` to **OFF**.

![Airflow Pause DAG](5.6-airflow-off.png)

---

### Step 2: Delete the Amazon Redshift Cluster

1. Open the **Amazon Redshift Console** and select the `redshift-ecom-dw` cluster.
2. Choose **Actions** → **Delete**.
3. If you do not need to retain the data, uncheck **Create final snapshot**, then confirm the deletion.

---

### Step 3: Clean Up the Amazon S3 Bucket

1. Open the **Amazon S3 Console** and select the `ecom-raw-data-lake-prod` bucket.
2. Choose **Empty** to remove all objects stored in the bucket.
3. After the bucket is empty, choose **Delete** to permanently remove the bucket.