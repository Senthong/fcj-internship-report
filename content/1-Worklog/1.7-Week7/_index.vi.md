---
title: "Worklog Tuần 7"
date: 2026-07-13
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Mục tiêu tuần 7:

* Chuyển toàn bộ tài nguyên AWS đã dùng (S3, Redshift Serverless, IAM, Glue) thành mã Terraform, để cả hệ thống có thể tái lập từ đầu.
* Đưa các tài nguyên đã tạo thủ công ở Tuần 2–4 vào Terraform state một cách nhất quán, không làm mất dữ liệu hiện có.
* Đóng gói môi trường phát triển cục bộ (Airflow + Postgres + dbt) bằng Docker Compose.
* Áp dụng các thực hành bảo mật và kiểm soát chi phí ở tầng hạ tầng (đặc quyền tối thiểu, chặn truy cập công khai, lifecycle rule của S3).

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Học Terraform cơ bản: provider, resource, variable, state, `plan`/`apply` <br> - Import bucket S3 và Redshift Serverless namespace đã tạo thủ công vào Terraform state (`terraform import`) để các thay đổi sau này đi qua mã nguồn thay vì thao tác trên console | 13/07/2026   | 13/07/2026      | Terraform AWS Provider docs |
| 3   | - Viết `aws_s3_bucket` + `aws_s3_bucket_lifecycle_configuration` (chuyển sang INTELLIGENT_TIERING sau 90 ngày, xoá sau 365 ngày) + `aws_s3_bucket_public_access_block`                       | 14/07/2026   | 14/07/2026      | |
| 4   | - Viết `aws_iam_role`/policy cho quyền Redshift truy cập S3 <br> - Viết `aws_redshiftserverless_namespace`/`workgroup` (8 RPU cơ sở, `publicly_accessible = false`) + security group chỉ mở cổng 5439 | 15/07/2026   | 15/07/2026      | |
| 5   | - Viết `aws_glue_catalog_database` + crawler <br> - Thiết lập `variables.tf` để cùng một bộ mã Terraform triển khai được cả `dev` và `prod` chỉ bằng cách đổi biến `environment`               | 16/07/2026   | 16/07/2026      | |
| 6   | - Viết `docker-compose.yml` dùng YAML anchor `x-airflow-common` (postgres, airflow-webserver, airflow-scheduler, airflow-init, dịch vụ dbt) <br> - **Kiểm thử:** `terraform destroy` + `terraform apply` từ đầu trên tài khoản/vùng AWS mới, xác nhận toàn bộ hạ tầng dựng lại giống hệt | 17/07/2026   | 18/07/2026 | |


### Kết quả đạt được tuần 7:

* Đưa toàn bộ hạ tầng AWS vào Terraform: S3 (bucket thô + lifecycle + chặn truy cập công khai), Redshift Serverless (namespace + workgroup + security group), IAM role cho quyền Redshift→S3, và Glue catalog + crawler — tổng cộng 4 nhóm tài nguyên.

* Đưa các tài nguyên đã tạo thủ công ở Tuần 2–4 vào Terraform state qua `terraform import`, nhờ vậy dự án không cần phá bỏ và mất dữ liệu đã nạp — một bài học quan trọng: hạ tầng dưới dạng mã nguồn không nhất thiết phải làm lại từ đầu, mà có thể là chính thức hoá những gì đang chạy tốt.

* Kiểm chứng khả năng tái lập trực tiếp: chạy `terraform destroy` rồi `terraform apply` trên một vùng AWS khác, xác nhận pipeline có thể trỏ vào bucket/workgroup mới tạo mà không cần thao tác thủ công trên console.

* Cài đặt kiểm soát chi phí ở tầng hạ tầng: lifecycle rule của S3 chuyển dữ liệu cũ hơn 90 ngày sang `INTELLIGENT_TIERING` và xoá sau 365 ngày; Redshift Serverless giới hạn 8 RPU cơ sở thay vì cluster provisioned kích thước cố định.

* Cài đặt kiểm soát bảo mật: `aws_s3_bucket_public_access_block` chặn toàn bộ truy cập công khai vào bucket thô; Redshift Serverless workgroup đặt `publicly_accessible = false`; security group chỉ mở cổng 5439 cho traffic nội bộ.

* Hoàn thành `docker-compose.yml` cho môi trường phát triển cục bộ, dùng YAML anchor `x-airflow-common` để tránh lặp lại cấu hình environment/volume ba lần giữa webserver, scheduler và container khởi tạo.

* Rút ra bài học Terraform thực tế: vì bucket S3 và tài nguyên Redshift Serverless đã tồn tại từ Tuần 2–4, bước tiếp theo tự nhiên sau "học Terraform" không phải là provision từ con số không, mà là import và chính thức hoá hạ tầng đang có — gần với cách IaC thường được áp dụng trên một hệ thống đang chạy trong thực tế.
