---
title: "5.2 - Yêu cầu chuẩn bị"
date: 2026-07-21
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## 1. Yêu cầu cấu hình phần cứng tối thiểu

Để đảm bảo môi trường phát triển chạy mượt mà khi khởi chạy đồng thời nhiều Docker container (Apache Airflow, PostgreSQL) và kết nối với đám mây AWS, máy tính cá nhân cần đáp ứng các thông số phần cứng tối thiểu sau:

| Thành phần | Cấu hình tối thiểu | Cấu hình khuyến nghị |
| :--- | :--- | :--- |
| **Hệ điều hành** | Windows 10/11 Pro/Enterprise (WSL2), macOS 12+, Ubuntu 20.04 LTS | Windows 11 Pro (WSL2), macOS Apple Silicon (M1/M2/M3), Ubuntu 22.04 LTS |
| **CPU** | 4 Cores / 8 Threads | 8 Cores / 16 Threads |
| **Bộ nhớ RAM** | 8 GB | 16 GB trở lên (đặc biệt cần thiết khi chạy Airflow) |
| **Dung lượng đĩa trống** | 20 GB SSD | 50 GB NVMe SSD |

---

## 2. Tài khoản AWS và Cấu hình Phân quyền (IAM)

Dự án sử dụng hai dịch vụ hạ tầng đám mây cốt lõi của AWS là Amazon S3 và Amazon Redshift. Trước khi bắt đầu, bạn cần chuẩn bị tài khoản và phân quyền truy cập.

### 2.1. Tài khoản AWS Active
* Tài khoản AWS đã được xác thực thanh toán và đang ở trạng thái hoạt động (Active).
* Quyền truy cập vào **AWS Management Console**.

### 2.2. Khởi tạo IAM User và Access Keys
* Tạo một IAM User chuyên dụng dành cho bài thực hành (ví dụ: `data-pipeline-admin`).
* Cấp quyền cho IAM User với các chính sách (Policies) tối thiểu:
  * `AmazonS3FullAccess`: Quyền tạo, đọc, ghi và xóa dữ liệu trên AWS S3 Bucket.
  * `AmazonRedshiftFullAccess`: Quyền tạo, quản lý cluster/serverless và thực thi query trên Amazon Redshift.
* Khởi tạo **Access Key ID** và **Secret Access Key** thuộc dạng *Programmatic Access* để cấu hình kết nối tự động từ Apache Airflow và AWS CLI.

![IAM User](5.2-aws-iam-keys.png)
### 2.3. Cấu hình Tài nguyên Đám mây
1. **Amazon S3 Bucket:**
   * Tạo một S3 Bucket mới (ví dụ: `ecom-data-lake-raw-uniquename`).
   * Chọn vùng (Region) cùng khu vực với Redshift (khuyến nghị: `ap-southeast-1` - Singapore).
   * Giữ nguyên cấu hình chặn truy cập công khai (*Block all public access*).
2. **Amazon Redshift Cluster / Serverless:**
   * Khởi tạo Redshift Serverless Workgroup hoặc Provisioned Cluster (định dạng node: `dc2.large` hoặc `ra3.xlplus`).
   * Cấu hình **Inbound Rules** trong **VPC Security Group** của Redshift: Cho phép lưu lượng truy cập qua cổng `5439` từ địa chỉ IP cá nhân của bạn hoặc từ môi trường chạy Airflow.

---

## 3. Môi trường Container và Đóng gói (Docker)

Toàn bộ hệ thống pipeline cục bộ được đóng gói và vận hành thông qua Docker nhằm đảm bảo tính đồng nhất về môi trường chạy giữa các máy tính khác nhau.

### 3.1. Cài đặt Docker Engine & Docker Compose
* Cài đặt **Docker Desktop** (trên Windows/macOS) hoặc **Docker Engine** kèm plugin `docker-compose-plugin` (trên Linux).
* Đảm bảo phiên bản Docker Compose tối thiểu từ `v2.20.0` trở lên.

### 3.2. Cấu hình dịch vụ trong Docker Compose
Dự án vận hành tập trung thông qua tệp `docker-compose.yml`, bao gồm các container chính:

1. **Airflow Webserver:** Cung cấp giao diện quản trị đồ họa (UI) tại cổng `8080`.
2. **Airflow Scheduler:** Chịu trách nhiệm quét DAGs, lập lịch và phân phối công việc cho các Worker.
3. **Airflow Metadata Database:** Sử dụng PostgreSQL làm cơ sở dữ liệu lưu trữ trạng thái của pipeline.
4. **Source Database (OLTP):** Thư viện PostgreSQL độc lập đóng vai trò là cơ sở dữ liệu giao dịch nguồn, chứa dữ liệu thô đầu vào (Olist E-Commerce dataset).

![Docker Containers Running](5.2-docker-ps.png)

---

## 4. Công cụ Lập trình và Quản lý Cơ sở dữ liệu

### 4.1. Môi trường Python Cục bộ (Local Python Environment)
* Cài đặt Python phiên bản `3.11` trở lên trên máy cục bộ để phục vụ phát triển code script và chạy dbt CLI trực tiếp khi cần kiểm thử.
* Cài đặt gói mở rộng dbt dành cho Redshift:
  ```bash
  pip install dbt-core dbt-redshift
### 4.2. Trình soạn thảo VS Code và các Extension bắt buộc
Sử dụng Visual Studio Code làm môi trường phát triển tích hợp (IDE). Cần cài đặt sẵn các tiện ích mở rộng (Extensions) sau:

Python (ms-python.python): Hỗ trợ IntelliSense, linting và debug code Python.

dbt Power User (innoverio.vscode-dbt-power-user): Hỗ trợ compiled query, xem đồ thị dòng chảy dữ liệu (Lineage Graph) và chạy nhanh dbt model.

Docker (ms-azuretools.vscode-docker): Quản lý container, images và theo dõi log trực tiếp trên IDE.

YAML (redhat.vscode-yaml): Kiểm tra cú pháp các file cấu hình docker-compose.yml, profiles.yml và schema.yml.

5. Danh mục kiểm tra mức độ sẵn sàng (Readiness Checklist)Trước khi tiến hành bước tiếp theo, hãy đảm bảo bạn đã tích chọn hoàn thành tất cả các mục trong bảng kiểm tra dưới đây:STTHạng mục kiểm traLệnh xác minh / Thao tácTrạng thái đạt1Docker Engine & Composedocker --version và docker compose versionĐạt2Các Container dự ándocker ps hiển thị đủ Airflow và PostgresĐạt3Quyền truy cập AWS CLIaws sts get-caller-identity trả về thông tin IAM UserĐạt4Kết nối S3 Bucketaws s3 ls s3://<ten-bucket-cua-ban> không báo lỗiĐạt5dbt Core CLIdbt --version hiển thị phiên bản dbt-core và dbt-redshiftĐạt6Giao diện Airflow UITruy cập http://localhost:8080 đăng nhập thành côngĐạt