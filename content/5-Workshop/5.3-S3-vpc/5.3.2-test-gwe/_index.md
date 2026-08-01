---
title: "5.3.2 - Verifying the Connection & Configuring the dbt Profile"
date: 2026-07-22
weight: 2
chapter: false
pre: " <b> 5.3.2 </b> "
---

### Verify the dbt Connection to Amazon Redshift

1. Open the `~/.dbt/profiles.yml` file (or the project's configuration file) and configure the Redshift connection details as shown below:

```yaml
ecom_pipeline:
  target: prod
  outputs:
    prod:
      type: redshift
      host: redshift-ecom-dw.xxxx.ap-southeast-1.redshift.amazonaws.com
      user: awsuser
      password: "{{ env_var('REDSHIFT_PASSWORD') }}"
      port: 5439
      dbname: dev
      schema: mart
      threads: 8
```

After saving the configuration, verify the connection by running:

```bash
dbt debug
```

If the configuration is correct, `dbt debug` should report that the connection to the Amazon Redshift cluster was established successfully.