---
title: "Worklog"
date: 2026-06-01
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

**Trong trang này**, tôi trình bày worklog thực tập tốt nghiệp, ghi lại quá trình thiết kế và xây dựng đề tài **E-Commerce Data Pipeline** — một hệ thống phân tích dữ liệu end-to-end trên nền tảng AWS (S3, Redshift Serverless, IAM, Glue), được điều phối bằng Apache Airflow, biến đổi dữ liệu bằng dbt, quản lý hạ tầng bằng Terraform, và kiểm thử tự động bằng GitHub Actions.

Quá trình thực tập kéo dài **8 tuần**, thực hiện theo phương pháp tiếp cận lặp: bắt đầu từ kiến thức nền tảng AWS, sau đó xây dựng pipeline theo từng tầng (Bronze → Silver → Gold), và kết thúc bằng việc quản lý hạ tầng dưới dạng mã nguồn cùng kiểm thử tự động.

**Tuần 1:** [Làm quen nền tảng AWS: Console, CLI, EC2, S3, IAM](1.1-week1/)

**Tuần 2:** [Tìm hiểu sâu S3, IAM và Amazon Redshift Serverless; khảo sát bộ dữ liệu Olist](1.2-week2/)

**Tuần 3:** [Xây dựng tầng Bronze: script Python ingestion (Kaggle → S3)](1.3-week3/)

**Tuần 4:** [Xây dựng tầng Silver: nạp dữ liệu vào Redshift bằng COPY, thiết kế DISTKEY/SORTKEY](1.4-week4/)

**Tuần 5:** [Xây dựng tầng Gold: mô hình staging & mart bằng dbt, kiểm thử chất lượng dữ liệu](1.5-week5/)

**Tuần 6:** [Điều phối pipeline bằng Apache Airflow (DAG 7 task, retry, quality gate)](1.6-week6/)

**Tuần 7:** [Quản lý hạ tầng bằng Terraform và môi trường phát triển cục bộ bằng Docker Compose](1.7-week7/)

**Tuần 8:** [CI/CD bằng GitHub Actions, kiểm thử end-to-end, đánh giá và hoàn thiện báo cáo](1.8-week8/)
