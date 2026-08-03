---
title: "Worklog Tuần 2"
date: 2026-06-08
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Mục tiêu tuần 2:

* Tìm hiểu sâu hơn 3 dịch vụ AWS mà pipeline phụ thuộc nhiều nhất: S3, IAM và Redshift Serverless.
* Hiểu kiến trúc phân tầng dữ liệu (Bronze–Silver–Gold) và xác định cách ánh xạ lên S3 + Redshift.
* Khảo sát chi tiết bộ dữ liệu Olist Brazilian E-Commerce: 9 tệp CSV có quan hệ, schema, và các vấn đề chất lượng dữ liệu.
* Phác thảo sơ đồ kiến trúc tổng thể của hệ thống trước khi bắt đầu viết code.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Tìm hiểu Amazon S3: bucket, object storage, prefix/partitioning, lifecycle policy, chặn truy cập công khai                                                                                 | 08/06/2026   | 08/06/2026      | AWS S3 User Guide |
| 3   | - Tìm hiểu IAM: user, role, policy, nguyên tắc đặc quyền tối thiểu <br> - **Thực hành:** tạo IAM Role để Redshift assume role đọc dữ liệu từ S3 (không dùng access key tĩnh)                | 09/06/2026   | 09/06/2026      | AWS IAM Documentation |
| 4   | - Tìm hiểu Amazon Redshift Serverless: RPU, workgroup/namespace, DISTKEY vs SORTKEY, lệnh COPY <br> - **Thực hành:** provision thủ công một Redshift Serverless workgroup nhỏ qua Console   | 10/06/2026   | 10/06/2026      | AWS Redshift Serverless docs |
| 5   | - Học khái niệm Data Warehouse: OLTP vs OLAP, kiến trúc phân tầng (Bronze/Silver/Gold), ETL vs ELT <br> - Quyết định: đề tài đi theo mô hình ELT (nạp thô vào Redshift trước, biến đổi bằng dbt sau) | 11/06/2026   | 11/06/2026      | Kimball, *The Data Warehouse Toolkit* |
| 6   | - Tải và khảo sát bộ dữ liệu Olist Brazilian E-Commerce (9 tệp CSV) <br> - Vẽ quan hệ giữa orders, customers, order_items, sellers, products, payments, reviews <br> - Phác thảo sơ đồ kiến trúc tổng thể của pipeline | 12/06/2026   | 13/06/2026 | <https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce> |


### Kết quả đạt được tuần 2:

* Hiểu S3 với vai trò tầng lưu trữ Bronze: cách phân vùng theo ngày (`year=/month=/day=`) hoạt động, và vì sao lifecycle rule quan trọng để kiểm soát chi phí.

* Tạo IAM Role đầu tiên để Redshift assume khi đọc dữ liệu từ S3 — hiểu vì sao cách này được ưu tiên hơn access key tĩnh (giảm rủi ro rò rỉ thông tin xác thực).

* Provision thủ công một Redshift Serverless workgroup và chạy thử query đầu tiên, để hiểu cơ chế co giãn theo RPU trước khi tự động hoá bằng Terraform ở các tuần sau.

* Nắm được sự khác biệt giữa DISTKEY (cách phân phối dòng dữ liệu lên các node) và SORTKEY (thứ tự sắp xếp vật lý trên đĩa), và vì sao chọn sai sẽ ảnh hưởng hiệu năng JOIN/lọc khi dữ liệu lớn.

* Hiểu kiến trúc phân tầng dữ liệu (medallion) và chốt thiết kế: Bronze = CSV thô trên S3, Silver = bảng thô trong schema `staging` của Redshift, Gold = bảng `mart` được mô hình hoá bằng dbt.

* Chọn ELT thay vì ETL: vì Redshift có thể xử lý song song (MPP), nạp dữ liệu thô trước rồi biến đổi bằng SQL ngay trong kho dữ liệu hiệu quả hơn so với biến đổi bên ngoài bằng một engine tính toán riêng.

* Nắm rõ toàn bộ bộ dữ liệu Olist: 9 tệp CSV bao phủ toàn bộ vòng đời đơn hàng (khách hàng → đơn hàng → order_items → thanh toán → đánh giá), cộng thêm bảng dịch danh mục sản phẩm (Bồ Đào Nha → Anh) và bảng toạ độ địa lý.

* Hoàn thành bản phác thảo đầu tiên của sơ đồ kiến trúc tổng thể (Kaggle → S3 → Redshift staging → dbt staging/mart), làm cơ sở cho Hình 3.1 trong báo cáo cuối kỳ.
