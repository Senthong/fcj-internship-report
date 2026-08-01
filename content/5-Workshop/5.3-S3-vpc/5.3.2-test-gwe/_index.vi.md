---
title : "5.3.2 - Kiểm tra Kết nối & dbt Profile"
date : 2026-07-22 
weight : 2
chapter : false
pre : " <b> 5.3.2 </b> "
---

### Kiểm tra kết nối dbt đến Redshift

1. Mở file `~/.dbt/profiles.yml` hoặc file config trong dự án và điền thông số kết nối:

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