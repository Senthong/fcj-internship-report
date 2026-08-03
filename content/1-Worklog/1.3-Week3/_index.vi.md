---
title: "Worklog Tuần 3"
date: 2026-06-15
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Mục tiêu tuần 3:

* Xây dựng tầng Bronze: script Python tải bộ dữ liệu Olist và tải lên S3 một cách an toàn, có thể chạy lại nhiều lần.
* Cài đặt idempotency: chạy script hai lần cho cùng một ngày không được tạo dữ liệu trùng lặp hoặc lãng phí thời gian.
* Cài đặt kiểm tra toàn vẹn dữ liệu (checksum MD5) và tệp manifest phục vụ giám sát.
* Thiết lập môi trường Python và cấu trúc thư mục dự án.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Dựng cấu trúc thư mục dự án (`ingestion/`, `dbt_project/`, `airflow/`, `infrastructure/`) <br> - Cài Python 3.11 virtualenv, `requirements.txt` (boto3, pandas, kaggle)                    | 15/06/2026   | 15/06/2026      | |
| 3   | - Cấu hình Kaggle API credentials <br> - Viết `download_olist_dataset()`: tải 9 tệp CSV từ Kaggle về thư mục tạm cục bộ                                                                      | 16/06/2026   | 16/06/2026      | Kaggle API docs |
| 4   | - Thiết kế sơ đồ phân vùng S3 theo ngày: `s3://{bucket}/olist/year=YYYY/month=MM/day=DD/` <br> - Viết `check_already_ingested()` cho idempotency: bỏ qua nếu phân vùng đã tồn tại (trừ khi có `--force`) | 17/06/2026   | 17/06/2026      | |
| 5   | - Viết `upload_to_s3()`: đính kèm checksum MD5, kích thước tệp, thời điểm tải lên vào metadata của object S3 <br> - Viết `validate_csv()`: kiểm tra sơ bộ (số dòng, null) trước khi upload    | 18/06/2026   | 18/06/2026      | |
| 6   | - Ghép nối thành hàm `ingest()` hoàn chỉnh <br> - Ghi `_manifest.json` sau mỗi lần chạy (log thành công/thất bại từng tệp) <br> - **Kiểm thử:** chạy 2 lần cho cùng một ngày, xác nhận lần thứ hai bị bỏ qua | 19/06/2026   | 20/06/2026 | |


### Kết quả đạt được tuần 3:

* Dựng xong cấu trúc repository tách bạch rõ trách nhiệm: `ingestion/`, `dbt_project/`, `airflow/dags/`, `infrastructure/terraform/`, khớp với kiến trúc đã thiết kế ở Tuần 2.

* Hoàn thành `ingestion/ingest_olist_to_s3.py`, với hàm cốt lõi `ingest(run_date, force=False)` thực hiện tuần tự:
  1. kiểm tra phân vùng của ngày đã tồn tại trên S3 chưa (idempotency),
  2. tải 9 tệp CSV Olist từ Kaggle,
  3. kiểm thử sơ bộ từng tệp,
  4. tải lên S3 kèm metadata MD5/kích thước/thời gian,
  5. ghi `_manifest.json` tóm tắt kết quả lần chạy.

* Xác nhận cơ chế idempotency hoạt động đúng: chạy lại `ingest()` cho một ngày đã ingest sẽ log `"already exists, skipping"` và trả về ngay, thay vì tải lại ~50MB CSV và upload lại lên S3.

* Hiểu vì sao metadata MD5 quan trọng: nó cho phép kiểm tra tính toàn vẹn tệp về sau mà không cần tải lại từ nguồn — điều này trở nên quan trọng khi pipeline chạy tự động theo lịch mà không có người giám sát.

* Chạy thử end-to-end thủ công thành công lần đầu: cả 9 tệp đều nằm đúng phân vùng ngày trên S3, manifest xác nhận 9/9 tệp tải lên thành công.

* Rút ra bài học thực tế về thiết kế idempotency: kiểm tra "phân vùng đã tồn tại chưa" tốn ít chi phí (chỉ một lệnh `list_objects` trên S3) hơn nhiều so với kiểm tra từng tệp — điều này có ý nghĩa khi bước này chạy hằng ngày bên trong Airflow.
