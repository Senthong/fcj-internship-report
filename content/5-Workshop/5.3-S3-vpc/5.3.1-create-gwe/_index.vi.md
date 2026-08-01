---
title: "5.3.1 - Khởi tạo S3 Bucket & Redshift Cluster"
weight: 1
---

### Bước 1: Tạo S3 Bucket chứa dữ liệu thô
1. Truy cập **AWS S3 Console** ➔ Chọn **Create bucket**.
2. Đặt tên Bucket: `ecom-raw-data-lake-prod`.
3. Giữ cấu hình mặc định và nhấn **Create bucket**.

![Create S3 Bucket](images/5.3.1-s3-bucket.png)
> 📸 **Gợi ý chụp ảnh 5.3.1a:** Chụp màn hình trang danh sách S3 Buckets hiển thị bucket `ecom-raw-data-lake-prod` đã tạo thành công.

---

### Bước 2: Tạo Amazon Redshift Cluster
1. Truy cập **Amazon Redshift Console** ➔ Chọn **Create cluster**.
2. Cấu hình thông số:
   * **Cluster identifier:** `redshift-ecom-dw`
   * **Database name:** `dev`
   * **Admin user:** `awsuser`
3. Tạo các schema cần thiết trên Redshift:
```sql
CREATE SCHEMA staging;
CREATE SCHEMA mart;