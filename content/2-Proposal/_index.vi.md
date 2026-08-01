---
title: "Bản đề xuất"
date: 2026-06-08
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

Tại phần này, tôi sẽ trình bày tổng quan về dự án **E-Commerce Data Pipeline**, mục tiêu, kiến trúc hệ thống, kế hoạch triển khai và những lợi ích dự kiến đạt được.

# E-Commerce Data Pipeline
## Xây dựng hệ thống ELT tự động trên AWS phục vụ phân tích dữ liệu thương mại điện tử

### 1. Tóm tắt điều hành

E-Commerce Data Pipeline là hệ thống xử lý dữ liệu được xây dựng nhằm tự động hóa quá trình thu thập, lưu trữ, chuyển đổi và phân tích dữ liệu thương mại điện tử. Dự án sử dụng bộ dữ liệu công khai **Brazilian E-Commerce (Olist Dataset)** và áp dụng kiến trúc ELT hiện đại trên nền tảng AWS.

Hệ thống sử dụng Amazon S3 làm vùng lưu trữ dữ liệu thô, Amazon Redshift làm kho dữ liệu trung tâm, Apache Airflow để điều phối pipeline, dbt để xây dựng các mô hình dữ liệu và Terraform để quản lý hạ tầng theo mô hình Infrastructure as Code. Ngoài ra, GitHub Actions được sử dụng để tự động kiểm tra mã nguồn mỗi khi có thay đổi.

Mục tiêu của dự án là xây dựng một pipeline dữ liệu hoàn chỉnh, có khả năng mở rộng, dễ bảo trì và đáp ứng nhu cầu phân tích dữ liệu phục vụ hoạt động kinh doanh.

---

## 2. Tuyên bố vấn đề

### Vấn đề hiện tại

Trong nhiều doanh nghiệp, dữ liệu giao dịch được lưu trữ ở nhiều bảng khác nhau và thường chỉ phục vụ cho hệ thống vận hành. Việc tổng hợp dữ liệu để tạo báo cáo kinh doanh thường được thực hiện thủ công, gây mất nhiều thời gian và dễ xảy ra sai sót.

Bên cạnh đó, dữ liệu chưa được chuẩn hóa khiến việc phân tích doanh thu, khách hàng hay hiệu quả bán hàng gặp nhiều khó khăn. Khi khối lượng dữ liệu tăng lên, các quy trình ETL thủ công càng trở nên khó bảo trì và mở rộng.

### Giải pháp

Dự án đề xuất xây dựng một hệ thống ELT tự động trên AWS.

Pipeline thực hiện các bước sau:

- Đưa dữ liệu gốc từ bộ dữ liệu Olist lên Amazon S3.
- Nạp dữ liệu từ S3 vào Amazon Redshift.
- Apache Airflow tự động điều phối toàn bộ quy trình xử lý.
- dbt thực hiện làm sạch, chuẩn hóa và xây dựng các mô hình phân tích.
- Tạo các bảng dữ liệu phục vụ báo cáo doanh thu, khách hàng và hiệu quả kinh doanh.
- Terraform tự động triển khai hạ tầng AWS.
- GitHub Actions tự động kiểm tra chất lượng mã nguồn.

### Lợi ích và hiệu quả

Hệ thống giúp tự động hóa hoàn toàn quy trình xử lý dữ liệu, giảm đáng kể thời gian thao tác thủ công và tăng tính nhất quán của dữ liệu.

Ngoài việc phục vụ mục đích học tập và nghiên cứu về Data Engineering, dự án còn có thể được mở rộng cho các hệ thống phân tích dữ liệu thực tế với quy mô lớn hơn.

---

## 3. Kiến trúc giải pháp

Kiến trúc hệ thống được xây dựng theo mô hình ELT hiện đại trên nền tảng AWS.

Luồng xử lý dữ liệu:

```
Olist Dataset
      │
      ▼
 Amazon S3
      │
      ▼
Amazon Redshift
      │
      ▼
Apache Airflow
      │
      ▼
      dbt
      │
      ▼
 Data Mart
```

*(Chèn sơ đồ kiến trúc tại đây)*

### Các công nghệ sử dụng

- **Amazon S3:** Lưu trữ dữ liệu thô.
- **Amazon Redshift:** Kho dữ liệu phục vụ phân tích.
- **Apache Airflow:** Điều phối và tự động hóa pipeline.
- **dbt:** Chuyển đổi và mô hình hóa dữ liệu.
- **Terraform:** Quản lý hạ tầng dưới dạng mã nguồn.
- **GitHub Actions:** Tự động kiểm tra mã nguồn.
- **Python:** Xử lý và nạp dữ liệu.
- **Docker Compose:** Triển khai môi trường phát triển.

### Thiết kế các thành phần

**Thu thập dữ liệu**

Dữ liệu từ bộ dữ liệu Olist được tải lên Amazon S3 bằng các chương trình Python.

**Kho dữ liệu**

Amazon Redshift lưu trữ dữ liệu gốc và dữ liệu sau khi chuyển đổi để phục vụ truy vấn phân tích.

**Điều phối quy trình**

Apache Airflow quản lý toàn bộ pipeline từ bước nạp dữ liệu đến thực thi các mô hình dbt.

**Chuyển đổi dữ liệu**

dbt xây dựng các lớp dữ liệu theo mô hình nhiều tầng:

- Staging Layer
    - Customers
    - Orders
    - Order Items
    - Order Payments
    - Order Reviews
    - Products
    - Sellers

- Mart Layer
    - Daily Revenue
    - Monthly Category Revenue
    - Seller Performance
    - Customer Cohort Analysis

**Quản lý hạ tầng**

Terraform triển khai các tài nguyên AWS, giúp quá trình triển khai có thể lặp lại và dễ bảo trì.

**CI/CD**

GitHub Actions tự động chạy kiểm tra mã nguồn khi có thay đổi trên GitHub.

---

## 4. Triển khai kỹ thuật

### Các giai đoạn triển khai

Dự án được chia thành bốn giai đoạn.

**Giai đoạn 1 – Nghiên cứu và thiết kế**

- Tìm hiểu kiến trúc ELT.
- Thiết kế mô hình dữ liệu.
- Lựa chọn các dịch vụ AWS.

**Giai đoạn 2 – Xây dựng hạ tầng**

- Cấu hình Terraform.
- Tạo Amazon S3.
- Tạo Amazon Redshift.
- Cấu hình IAM.

**Giai đoạn 3 – Xây dựng Pipeline**

- Phát triển chương trình nạp dữ liệu.
- Xây dựng Airflow DAG.
- Nạp dữ liệu vào Redshift.
- Xây dựng các mô hình dbt.

**Giai đoạn 4 – Kiểm thử và triển khai**

- Kiểm thử toàn bộ pipeline.
- Kiểm tra các mô hình dbt.
- Thiết lập GitHub Actions.
- Hoàn thiện tài liệu.

### Yêu cầu kỹ thuật

**Ngôn ngữ lập trình**

- Python
- SQL

**Công cụ**

- Apache Airflow
- dbt
- Docker
- Terraform
- Git
- GitHub Actions

**Dịch vụ AWS**

- Amazon S3
- Amazon Redshift
- IAM

---

## 5. Lộ trình và các mốc triển khai

### Thời gian thực hiện

**Tháng 1**

- Nghiên cứu kiến trúc.
- Thiết kế pipeline.
- Chuẩn bị dữ liệu.

**Tháng 2**

- Xây dựng hạ tầng AWS.
- Hoàn thiện chương trình nạp dữ liệu.
- Xây dựng Airflow.

**Tháng 3**

- Hoàn thiện dbt.
- Kiểm thử.
- Viết tài liệu.
- Triển khai hệ thống.

---

## 6. Ước tính chi phí

Dự án ưu tiên sử dụng AWS Free Tier và các tài nguyên có chi phí thấp.

### Chi phí hạ tầng

- Amazon S3
- Amazon Redshift
- Data Transfer
- IAM

Ước tính chi phí khoảng **5–10 USD/tháng**, tùy theo dung lượng dữ liệu và thời gian sử dụng Redshift.

---

## 7. Đánh giá rủi ro

### Ma trận rủi ro

- Cấu hình sai tài nguyên AWS: Ảnh hưởng trung bình, xác suất trung bình.
- Lỗi khi thực thi Airflow DAG: Ảnh hưởng trung bình, xác suất trung bình.
- Sai sót trong quá trình chuyển đổi dữ liệu: Ảnh hưởng cao, xác suất thấp.
- Phát sinh chi phí AWS ngoài dự kiến: Ảnh hưởng thấp, xác suất thấp.

### Biện pháp giảm thiểu

- Quản lý hạ tầng bằng Terraform.
- Thiết lập kiểm tra tự động với GitHub Actions.
- Sử dụng dbt Test để kiểm tra chất lượng dữ liệu.
- Theo dõi quá trình thực thi Airflow.

### Kế hoạch dự phòng

- Chạy lại các DAG bị lỗi.
- Khôi phục hạ tầng bằng Terraform.
- Nạp lại dữ liệu từ Amazon S3 khi cần thiết.

---

## 8. Kết quả kỳ vọng

### Kết quả kỹ thuật

Sau khi hoàn thành, hệ thống sẽ cung cấp một pipeline ELT hoàn chỉnh bao gồm:

- Tự động thu thập dữ liệu.
- Lưu trữ trên AWS.
- Điều phối bằng Apache Airflow.
- Chuyển đổi dữ liệu bằng dbt.
- Quản lý hạ tầng bằng Terraform.
- Tự động kiểm tra mã nguồn với GitHub Actions.

### Giá trị lâu dài

Dự án là nền tảng thực hành Data Engineering theo tiêu chuẩn hiện đại và có thể tiếp tục mở rộng trong tương lai với các chức năng như:

- Incremental Loading.
- Data Quality Monitoring.
- Dashboard trực quan bằng Power BI hoặc Amazon QuickSight.
- Tích hợp Machine Learning và các hệ thống phân tích dữ liệu nâng cao.