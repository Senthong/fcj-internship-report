---
title: "Triển khai tầng lưu trữ và kho dữ liệu trên AWS"
date: 2026-07-22
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

Phần này trình bày quá trình triển khai hai thành phần hạ tầng cốt lõi của hệ thống trên AWS: tầng lưu trữ dữ liệu thô (Amazon S3) và tầng kho dữ liệu (Amazon Redshift Serverless), cùng với việc thiết lập đường kết nối riêng giữa hai thành phần này thông qua **Gateway VPC Endpoint**.

Một điểm quan trọng cần làm rõ trước khi đi vào chi tiết: theo cấu hình được định nghĩa trong Terraform (`infrastructure/terraform/main.tf`), Redshift Serverless Workgroup của dự án được khởi tạo với thuộc tính `publicly_accessible = false`, đồng nghĩa với việc Workgroup này chỉ có địa chỉ mạng riêng (private) nằm trong các subnet riêng tư (private subnet) của VPC, không có địa chỉ IP công khai và không thể truy cập trực tiếp từ Internet. Đây là một lựa chọn thiết kế đúng đắn về mặt bảo mật - kho dữ liệu chứa thông tin giao dịch của khách hàng và người bán không nên có bất kỳ đường truy cập trực tiếp nào từ Internet công cộng.

Tuy nhiên, chính lựa chọn này lại đặt ra một yêu cầu kỹ thuật cần giải quyết: khi Redshift thực thi câu lệnh `COPY` để nạp dữ liệu từ Amazon S3 (một dịch vụ nằm ngoài phạm vi VPC theo mặc định), luồng dữ liệu này cần có một đường đi hợp lệ mà không được phép đi qua Internet công cộng. Nếu không có cấu hình bổ sung, câu lệnh `COPY` sẽ thất bại hoặc buộc phải định tuyến qua NAT Gateway - một giải pháp vừa phát sinh thêm chi phí, vừa không cần thiết vì đích đến (Amazon S3) là một dịch vụ của chính AWS. Giải pháp phù hợp cho tình huống này là sử dụng **Gateway VPC Endpoint** cho Amazon S3: một cơ chế định tuyến ở cấp độ Route Table, cho phép các tài nguyên nằm trong VPC (bao gồm Redshift Serverless) truy cập Amazon S3 hoàn toàn thông qua mạng backbone nội bộ của AWS, không phát sinh chi phí truyền dữ liệu qua NAT Gateway và không đi qua Internet công cộng dù chỉ một chặng.

Phần 5.3 được chia thành hai mục nhỏ, tương ứng với hai giai đoạn triển khai thực tế:

- [5.3.1 - Khởi tạo hạ tầng lưu trữ và kết nối riêng tới Amazon S3](5.3.1-create-gwe/): trình bày việc khởi tạo Amazon S3 Bucket, Amazon Redshift Serverless, và cấu hình Gateway VPC Endpoint.
- [5.3.2 - Kiểm tra kết nối và cấu hình dbt Profile](5.3.2-test-gwe/): trình bày việc xác minh đường kết nối đã hoạt động đúng thông qua việc cấu hình và kiểm thử kết nối dbt tới Redshift.
