---
title: "Worklog Tuần 5"
date: 2026-06-29
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Mục tiêu tuần 5:

* Xây dựng tầng Gold bằng dbt: 7 mô hình staging (view) làm sạch dữ liệu thô, và 4 mô hình mart (table) áp dụng logic nghiệp vụ.
* Viết 46 bài kiểm thử chất lượng dữ liệu (17 ở nguồn, 29 ở tầng model) làm cổng chất lượng trước khi dữ liệu đến tầng mart.
* Thiết kế macro `generate_schema_name` để tách biệt schema dev/prod.
* Hiểu đồ thị lineage `ref()`/`source()` của dbt và nguyên tắc DRY mà nó mang lại.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Cài dbt-core + dbt-redshift, thiết lập `profiles.yml` (target dev/prod) <br> - Cài package `dbt_utils` và `audit_helper` <br> - Khai báo `sources.yml` cho 8 bảng thô Redshift, kèm ràng buộc freshness (cảnh báo 25h, lỗi 49h) | 29/06/2026   | 29/06/2026      | docs.getdbt.com |
| 3   | - Viết 7 mô hình staging (view): `stg_customers`, `stg_orders`, `stg_order_items`, `stg_order_payments`, `stg_order_reviews`, `stg_products`, `stg_sellers` — ép kiểu, chuẩn hoá, cột dẫn xuất (`is_delivered`, `is_late_delivery`) | 30/06/2026   | 01/07/2026      | |
| 4   | - Viết mô hình mart `revenue_daily` và `category_revenue_monthly` (finance) dùng `ref()` để JOIN các mô hình staging                                                                        | 02/07/2026   | 02/07/2026      | |
| 5   | - Viết mô hình mart `seller_performance` (operations, công thức seller_score: 50% đánh giá + 50% giao đúng hạn) và `fct_customer_cohorts` (customers, giữ chân theo cohort)                | 03/07/2026   | 03/07/2026      | |
| 6   | - Viết 46 bài test dbt trong các tệp `schema.yml` (unique, not_null, accepted_values, `dbt_utils.expression_is_true`) <br> - Viết macro `generate_schema_name` tách schema dev/prod <br> - **Kiểm thử:** chạy `dbt run` + `dbt test`, xác nhận cả 11 model build thành công và tất cả test pass | 04/07/2026   | 05/07/2026 | |


### Kết quả đạt được tuần 5:

* Hoàn thành cả 7 mô hình staging dưới dạng view — không tốn chi phí lưu trữ, luôn phản ánh dữ liệu thô mới nhất. `stg_orders` là mô hình phức tạp nhất: tính `is_delivered`, `is_late_delivery`, `days_late` một lần duy nhất, nhờ vậy các mô hình mart phía sau không phải lặp lại logic `CASE WHEN` đó.

* Hoàn thành cả 4 mô hình mart dưới dạng table: `revenue_daily`, `category_revenue_monthly`, `seller_performance`, `fct_customer_cohorts` — mỗi bảng có grain rõ ràng (lần lượt: 1 dòng/ngày, 1 dòng/tháng+danh mục, 1 dòng/người bán, 1 dòng/cohort×tháng thứ n).

* Thiết kế công thức `seller_score` trong `seller_performance` (50% điểm đánh giá trung bình + 50% tỉ lệ giao đúng hạn) — ví dụ cụ thể về việc gộp nhiều tín hiệu thô thành một chỉ số scorecard phục vụ vận hành.

* Viết và chạy toàn bộ 46 bài kiểm thử chất lượng dữ liệu: 17 ở tầng nguồn (freshness, not_null trên bảng thô) và 29 ở tầng model (unique cho khoá chính, `accepted_values` cho các cột enum như `order_status`, kiểm tra miền giá trị như `seller_score BETWEEN 0 AND 100`).

* Xác nhận sơ đồ lineage của dbt (`dbt docs generate`) khớp với thiết kế ở Tuần 2: `stg_orders` và `stg_order_items` được 3–4 mô hình mart tham chiếu mỗi mô hình — minh chứng thực tế rõ ràng cho nguyên tắc DRY trong dbt.

* Xây dựng macro `generate_schema_name` để target `prod` tạo schema sạch (`staging`, `mart`), còn target `dev` tự động thêm tiền tố (`analytics_dev__staging`), tránh xung đột schema giữa nhiều nhà phát triển trên cùng một cụm Redshift.

* Phát hiện và xử lý vấn đề chất lượng dữ liệu thật đầu tiên: `dbt test` báo lỗi một vài dòng có `item_price` âm, dẫn đến việc thêm test `expression_is_true: ">= 0"` vào `stg_order_items` — bài học thực tế trực tiếp về lý do tồn tại của quality gate.
