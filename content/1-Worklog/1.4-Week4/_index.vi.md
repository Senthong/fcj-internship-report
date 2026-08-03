---
title: "Worklog Tuần 4"
date: 2026-06-22
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Mục tiêu tuần 4:

* Xây dựng tầng Silver: nạp 9 tệp CSV thô từ S3 vào 8 bảng thô trong schema `staging` của Redshift.
* Thiết kế DDL cho từng bảng thô, chọn DISTKEY/SORTKEY riêng dựa trên đặc điểm truy vấn dự kiến.
* Cài đặt chiến lược Truncate-and-Load, bọc trong một transaction nguyên tử duy nhất cho cả 8 bảng.
* Ưu tiên xác thực bằng IAM Role cho lệnh `COPY` thay vì access key tĩnh.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Thiết kế cấu trúc `TableSpec` (csv_file, table_name, dist_key, sort_key, ddl) để khai báo schema + tối ưu vật lý ở cùng một chỗ                                                            | 22/06/2026   | 22/06/2026      | |
| 3   | - Viết DDL cho toàn bộ 8 bảng thô (`raw_orders`, `raw_order_items`, `raw_customers`, `raw_sellers`, `raw_products`, `raw_order_payments`, `raw_order_reviews`, `raw_geolocation`)            | 23/06/2026   | 23/06/2026      | |
| 4   | - Chọn DISTKEY/SORTKEY cho từng bảng: `raw_orders` → DISTKEY(order_id)/SORTKEY(order_purchase_timestamp) vì đây là bảng hay JOIN nhất, hay lọc theo thời gian nhất; bảng chiều nhỏ → DISTSTYLE ALL | 24/06/2026   | 24/06/2026      | AWS Redshift best practices |
| 5   | - Cài đặt `truncate_and_load()`: CREATE TABLE IF NOT EXISTS → TRUNCATE → COPY FROM S3 <br> - Cài đặt `_auth_clause()` ưu tiên `REDSHIFT_IAM_ROLE_ARN` thay vì access key/secret key            | 25/06/2026   | 25/06/2026      | |
| 6   | - Bọc toàn bộ 8 lần nạp bảng trong một transaction `autocommit=False` duy nhất, có commit/rollback <br> - **Kiểm thử:** cố tình làm hỏng một dòng CSV để kích hoạt `MAXERROR`, xác nhận cả 8 bảng đều rollback cùng nhau | 26/06/2026   | 27/06/2026 | |


### Kết quả đạt được tuần 4:

* Hoàn thành `ingestion/load_to_redshift.py`, với 8 định nghĩa `TableSpec` gộp chung schema, DISTKEY và SORTKEY — nhờ vậy lý do của mỗi lựa chọn tối ưu nằm ngay cạnh DDL thay vì rải rác nhiều nơi.

* Hiểu rõ sự khác biệt thực tế giữa DISTKEY và SORTKEY qua thực hành: sau khi nạp `raw_orders` với `DISTKEY(order_id)`, các phép JOIN với `raw_order_items` (cũng khoá theo `order_id`) không cần bước broadcast/redistribute, vì plan `EXPLAIN` của Redshift không xuất hiện bước `DS_BCAST_INNER`.

* Cài đặt chiến lược Truncate-and-Load: vì `staging` chỉ là tầng trung gian (không lưu lịch sử), mỗi lần chạy an toàn thay thế dữ liệu ngày hôm trước thay vì tích luỹ trùng lặp.

* Kiểm chứng tính nguyên tử của transaction: cố tình chèn một dòng CSV không hợp lệ để vượt ngưỡng `MAXERROR` của `COPY`, xác nhận qua `conn.rollback()` rằng không bảng nào trong 8 bảng giữ lại dữ liệu chưa hoàn chỉnh — schema staging vẫn ở trạng thái nhất quán trước đó.

* Thiết lập xác thực bằng IAM Role (`REDSHIFT_IAM_ROLE_ARN`) làm phương án chính cho `COPY`, chỉ giữ access key/secret key làm phương án dự phòng — khớp với nguyên tắc đặc quyền tối thiểu đã học ở Tuần 2.

* Chạy thành công end-to-end lần đầu Bronze → Silver: S3 → 8 bảng trong schema `staging` của Redshift, đối chiếu số dòng với CSV nguồn để xác nhận không mất dữ liệu.
