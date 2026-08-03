---
title: "Worklog Tuần 6"
date: 2026-07-06
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Mục tiêu tuần 6:

* Điều phối toàn bộ pipeline (ingestion → nạp Redshift → dbt staging → quality gate → dbt mart → test → notify) bằng một DAG Apache Airflow duy nhất.
* Cài đặt quality gate: mô hình mart không được chạy nếu test staging thất bại.
* Cấu hình retry với exponential backoff và `max_active_runs=1` để tránh các lần chạy chồng lấn.
* Hiểu đủ sâu các khái niệm cốt lõi của Airflow (DAG, Operator, Scheduler, Executor) để có thể giải thích từng quyết định thiết kế trong báo cáo.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Học các khái niệm cốt lõi của Airflow: DAG, Operator (Python/Bash), Scheduler, Executor (Local/Celery/Kubernetes), XCom, retry & SLA                                                       | 06/07/2026   | 06/07/2026      | Apache Airflow docs |
| 3   | - Cài Airflow 2.8.1 cục bộ (ban đầu qua pip, để lặp nhanh hơn thay vì build lại Docker mỗi lần) <br> - Viết `DEFAULT_ARGS` với 2 lần retry, exponential backoff (5 phút → tối đa 30 phút)     | 07/07/2026   | 07/07/2026      | |
| 4   | - Viết task `ingest_olist_to_s3` và `load_staging_to_redshift` bằng `PythonOperator`, kết nối với các hàm đã viết ở Tuần 3–4 <br> - Viết `on_failure_callback=notify_failure` để ghi log chi tiết khi thất bại | 08/07/2026   | 08/07/2026      | |
| 5   | - Viết 4 task `BashOperator` gọi CLI dbt (`dbt run --select staging`, `dbt test --select staging`, `dbt run --select mart`, `dbt test --select mart`), truyền thông tin kết nối Redshift qua `env` | 09/07/2026   | 09/07/2026      | |
| 6   | - Nối 7 task theo chuỗi tuyến tính bằng `>>`, đặt `dbt_test_staging` làm quality gate trước `dbt_run_mart` <br> - Đặt `schedule_interval="0 2 * * *"`, `max_active_runs=1`, `catchup=False` <br> - **Kiểm thử:** trigger thủ công DAG chạy end-to-end, sau đó cố tình cho một test staging thất bại và xác nhận `dbt_run_mart` không bao giờ chạy | 10/07/2026   | 11/07/2026 | |


### Kết quả đạt được tuần 6:

* Hoàn thành `airflow/dags/ecom_full_pipeline_dag.py`: DAG tuyến tính 7 task — `ingest_olist_to_s3 → load_staging_to_redshift → dbt_run_staging → dbt_test_staging → dbt_run_mart → dbt_test_mart → notify_success`.

* Xác nhận quality gate hoạt động đúng thiết kế: cố tình đưa vào dữ liệu xấu để một assertion trong `dbt_test_staging` thất bại, xác nhận DAG dừng lại tại đó và `dbt_run_mart` không bao giờ được kích hoạt — nên không bảng mart nào có thể bị cập nhật bằng dữ liệu staging chưa được xác thực.

* Cấu hình và kiểm thử cơ chế retry: mô phỏng lỗi kết nối Redshift tạm thời, quan sát Airflow retry với exponential backoff 5 phút (giới hạn tối đa 30 phút) thay vì thất bại ngay lập tức — phù hợp với các lỗi tạm thời như nghẽn mạng hoặc giới hạn tốc độ.

* Hiểu vì sao `max_active_runs=1` quan trọng: nếu không có nó, một lần chạy DAG chạy chậm vẫn có thể đang thực thi trong khi lần chạy theo lịch của ngày hôm sau bắt đầu, cả hai cùng ghi vào schema `staging` một lúc.

* Nắm được sự khác biệt thực tế giữa `PythonOperator` (gọi hàm trực tiếp, phù hợp cho các bước ingestion/load đã tồn tại sẵn dưới dạng hàm Python có thể import) và `BashOperator` (phù hợp để điều khiển CLI dbt mà không cần viết lại logic thực thi của dbt bằng Python).

* Chạy DAG theo lịch cho nhiều ngày liên tiếp (backfill) để xác nhận idempotency giữ vững end-to-end: chạy lại nhiều lần cho cùng một ngày logic không tạo ra dòng dữ liệu trùng lặp trong Redshift hay các bảng mart.

* Lần đầu chạy toàn bộ pipeline thành công hoàn toàn tự động, không cần can thiệp, từ S3 đến bảng mart — một cột mốc quan trọng trước khi chuyển sang phần hạ tầng dưới dạng mã nguồn.
