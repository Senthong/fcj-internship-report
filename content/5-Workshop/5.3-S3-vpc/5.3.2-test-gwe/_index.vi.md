---
title: "5.3.2 - Kiểm tra kết nối và cấu hình dbt Profile"
date: 2026-07-22
weight: 2
chapter: false
pre: " <b> 5.3.2 </b> "
---

Sau khi hạ tầng lưu trữ, kho dữ liệu và đường kết nối riêng qua Gateway VPC Endpoint đã được thiết lập ở mục 5.3.1, bước tiếp theo là xác minh rằng toàn bộ đường kết nối này hoạt động đúng như thiết kế trước khi tiến hành xây dựng các model dbt ở phần 5.4. Việc xác minh được thực hiện gián tiếp thông qua dbt: nếu dbt kết nối thành công tới Redshift và một câu lệnh `COPY` từ S3 chạy không phát sinh lỗi mạng, có thể kết luận rằng cả kết nối tới Redshift lẫn đường định tuyến qua Gateway Endpoint đều đang hoạt động chính xác.

## 1. Cấu hình `profiles.yml`

dbt sử dụng tệp `profiles.yml` để lưu trữ thông tin kết nối đến kho dữ liệu, tách biệt hoàn toàn với mã nguồn các model (giúp tránh việc vô tình đưa thông tin xác thực lên hệ thống quản lý phiên bản). Tệp cấu hình của dự án định nghĩa hai môi trường đích (target) riêng biệt - `dev` và `prod` - dùng chung một profile tên `ecom_pipeline`:

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
      ra3_node: true       # cho phép truy vấn liên cơ sở dữ liệu (cross-database query)
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

Toàn bộ giá trị nhạy cảm (host, user, password) đều được đọc từ biến môi trường thông qua hàm `env_var()` của dbt, thay vì ghi trực tiếp vào tệp cấu hình - đây là cách làm nhất quán với việc quản lý biến môi trường đã trình bày ở phần 5.2. Tham số `ra3_node: true` được bật vì Workgroup Serverless của dự án thuộc dòng RA3, cho phép thực hiện truy vấn liên cơ sở dữ liệu khi cần thiết. Tham số `threads` được đặt khác nhau giữa hai môi trường (4 luồng cho `dev`, 8 luồng cho `prod`) nhằm cân bằng giữa tốc độ xử lý và mức tiêu thụ RPU của Workgroup trong từng giai đoạn - môi trường phát triển ưu tiên tiết kiệm tài nguyên trong khi môi trường sản xuất ưu tiên tốc độ.

Một điểm đáng chú ý khác là schema đích khác nhau giữa hai môi trường (`analytics_dev` cho `dev`, `analytics` cho `prod`); cơ chế đứng sau sự khác biệt này được trình bày chi tiết trong macro `generate_schema_name.sql` ở phần 5.5 của chương.

## 2. Kiểm tra kết nối bằng lệnh `dbt debug`

Sau khi lưu tệp cấu hình, kết nối được xác minh bằng lệnh sau, thực thi từ thư mục `dbt_project/`:

```bash
dbt debug
```

Lệnh này thực hiện tuần tự các bước kiểm tra: đọc và xác thực cú pháp tệp `profiles.yml`, phân giải các biến môi trường, mở kết nối TCP tới Redshift qua cổng `5439`, và thực thi một câu truy vấn kiểm tra đơn giản. Nếu toàn bộ hạ tầng ở phần 5.3.1 được cấu hình đúng, kết quả trả về xác nhận cả bốn hạng mục kiểm tra (Connection, Profile, Debug, All checks) đều ở trạng thái thành công (`PASS`).

Trong trường hợp `dbt debug` báo lỗi timeout khi thử kết nối, nguyên nhân phổ biến nhất không nằm ở cấu hình dbt mà nằm ở Security Group của Redshift (không cho phép lưu lượng từ máy đang chạy dbt) hoặc do máy đang chạy dbt không nằm trong cùng VPC/không có đường VPN/Bastion host để tiếp cận subnet riêng tư chứa Redshift - đây cũng là lý do vì sao trong môi trường Docker Compose của dự án, dbt được chạy từ bên trong cùng mạng Docker với Airflow, thay vì từ máy cá nhân kết nối trực tiếp.

## 3. Xác minh gián tiếp Gateway VPC Endpoint thông qua câu lệnh `COPY`

Việc `dbt debug` thành công mới chỉ xác nhận kết nối điều khiển (control plane) tới Redshift hoạt động bình thường, chưa xác nhận được đường dữ liệu (data plane) từ Redshift đến S3 qua Gateway Endpoint. Để xác minh đầy đủ, một câu lệnh `COPY` thử nghiệm được thực thi trực tiếp trong Query Editor v2, nạp thử một tệp mẫu từ bucket `ecom-pipeline-raw-prod` vào một bảng tạm trong schema `staging`:

```sql
COPY staging.raw_customers_test
FROM 's3://ecom-pipeline-raw-prod/olist/year=2024/month=01/day=15/olist_customers_dataset.csv'
IAM_ROLE 'arn:aws:iam::<account-id>:role/ecom-pipeline-redshift-s3-prod'
REGION 'ap-southeast-2'
FORMAT AS CSV
IGNOREHEADER 1;
```

Câu lệnh này chạy thành công và trả về số dòng đã nạp là bằng chứng cho thấy: (1) IAM Role gắn với Namespace có đủ quyền đọc bucket; (2) đường mạng từ Redshift đến S3 thông suốt, tức là Gateway VPC Endpoint và Route Table liên quan đã được cấu hình chính xác ở mục 5.3.1. Bảng tạm `raw_customers_test` được xóa ngay sau bước kiểm tra này bằng lệnh `DROP TABLE`, vì đây chỉ là bước xác minh hạ tầng, chưa phải bước nạp dữ liệu chính thức - việc nạp dữ liệu đầy đủ vào các bảng `staging` được trình bày chi tiết ở phần 5.4.1.

> **Ghi chú hình ảnh cần bổ sung:** Chèn hai ảnh chụp màn hình: (1) kết quả chạy lệnh `dbt debug` trong terminal, thể hiện đầy đủ bốn dòng kết quả kiểm tra đều ở trạng thái `PASS` và dòng thông báo cuối cùng "All checks passed!"; (2) kết quả thực thi câu lệnh `COPY` thử nghiệm trong Query Editor v2 của Redshift, thể hiện số dòng đã nạp thành công và không có thông báo lỗi ở panel kết quả.