---
title: "Worklog Tuần 1"
date: 2026-06-01
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Mục tiêu tuần 1:

* Kết nối, làm quen với các thành viên First Cloud AI Journey; đọc nội quy và định dạng báo cáo thực tập.
* Hiểu các nhóm dịch vụ AWS cơ bản (Compute, Storage, Networking, Database) và cách dùng Console & CLI.
* Nắm vững EC2 cơ bản, vì hầu hết các dịch vụ sau này (Redshift, Airflow chạy trên Docker) đều dùng chung khái niệm networking/IAM.
* Bắt đầu xác định phạm vi đề tài tốt nghiệp: hệ thống data pipeline end-to-end cho thương mại điện tử trên AWS.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Làm quen với các thành viên FCAJ <br> - Đọc và lưu ý các nội quy, quy định tại đơn vị thực tập <br> - Trao đổi với GVHD về hướng đề tài tốt nghiệp                                        | 01/06/2026   | 01/06/2026      |
| 3   | - Tìm hiểu AWS và các loại dịch vụ <br>&emsp; + Compute <br>&emsp; + Storage <br>&emsp; + Networking <br>&emsp; + Database <br>&emsp; + ... <br>                                            | 02/06/2026   | 02/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 4   | - Tạo AWS Free Tier account <br> - Tìm hiểu AWS Console & AWS CLI <br> - **Thực hành:** <br>&emsp; + Tạo AWS account <br>&emsp; + Cài AWS CLI & cấu hình <br> &emsp; + Cách sử dụng AWS CLI | 03/06/2026   | 03/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - Tìm hiểu EC2 cơ bản: <br>&emsp; + Instance types <br>&emsp; + AMI <br>&emsp; + EBS <br>&emsp; + ... <br> - Các cách remote SSH vào EC2 <br> - Tìm hiểu Elastic IP   <br>                  | 04/06/2026   | 05/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 6   | - **Thực hành:** <br>&emsp; + Tạo EC2 instance <br>&emsp; + Kết nối SSH <br>&emsp; + Gắn EBS volume <br> - Khảo sát bộ dữ liệu Olist Brazilian E-Commerce trên Kaggle để xác nhận tính khả thi của đề tài | 05/06/2026   | 05/06/2026      | <https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce> |


### Kết quả đạt được tuần 1:

* Hiểu AWS là gì và nắm được các nhóm dịch vụ cơ bản:
  * Compute
  * Storage
  * Networking
  * Database

* Đã tạo và cấu hình AWS Free Tier account thành công.

* Làm quen với AWS Management Console và biết cách tìm, truy cập, sử dụng dịch vụ từ giao diện web.

* Cài đặt và cấu hình AWS CLI trên máy tính bao gồm:
  * Access Key
  * Secret Key
  * Region mặc định

* Sử dụng AWS CLI để thực hiện các thao tác cơ bản như:
  * Kiểm tra thông tin tài khoản & cấu hình
  * Lấy danh sách region
  * Xem dịch vụ EC2
  * Tạo và quản lý key pair
  * Kiểm tra thông tin dịch vụ đang chạy

* Tạo EC2 instance, kết nối qua SSH và gắn EBS volume như một bài thực hành.

* Có khả năng kết nối giữa giao diện web và CLI để quản lý tài nguyên AWS song song.

* Xác nhận đề tài tốt nghiệp: xây dựng hệ thống data pipeline end-to-end phân tích dữ liệu thương mại điện tử trên AWS, sử dụng bộ dữ liệu công khai Olist Brazilian E-Commerce (hơn 100.000 đơn hàng, giai đoạn 2016–2018).
