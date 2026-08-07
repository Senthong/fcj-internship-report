---
title: "Các bài blog đã đăng"
date: 2026-06-01
weight: 3
chapter: false
pre: " <b> 3. </b> "
---


# Các bài blog trên AWS Study Group

Kho lưu trữ này bao gồm bộ sưu tập các bài blog mà tôi đã xuất bản trên [AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj), tập trung vào kiến trúc cloud-native, các giải pháp serverless, tối ưu hạ tầng AWS và các kinh nghiệm thực tế trong quá trình xây dựng ứng dụng trên nền tảng đám mây.

### [Blog 1 - AWS Glue (PySpark) và AWS Lambda + Polars: Khi Serverless vượt trội hơn Spark](3-BlogsPosted/3.1-Blog1)

Bài viết này so sánh hai kiến trúc ETL hiện đại trên AWS thông qua một bài kiểm thử thực tế: **AWS Glue (PySpark)** và **AWS Lambda + Polars được điều phối bởi AWS Step Functions**. Nội dung phân tích hiệu năng, chi phí, khả năng mở rộng và những đánh đổi về mặt kiến trúc, đồng thời cho thấy cách một thiết kế serverless gọn nhẹ có thể vượt trội đáng kể so với các cụm Spark truyền thống đối với các pipeline xử lý dữ liệu từ hàng GB đến hàng TB.

Đọc bài viết đầy đủ trên **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233435944088032/)**.

![alt text](image-1.png)

### [Blog 2 - Truy tìm và tiêu diệt các tài nguyên "Zombies" trên AWS](3-BlogsPosted/3.2-Blog2)

Bài viết chia sẻ trải nghiệm thực tế trong việc xác định và loại bỏ các **tài nguyên "Zombie"** trên AWS — những tài nguyên mồ côi vẫn tiếp tục phát sinh chi phí ngay cả khi các EC2 Instance liên quan đã được dừng hoặc xóa. Bài viết đề cập đến các "thủ phạm" phổ biến gây lãng phí chi phí như EBS Volume không được gắn, Elastic IP không sử dụng, Snapshot/AMI cũ, Load Balancer không hoạt động và NAT Gateway không sử dụng, cùng với các lệnh AWS CLI thực tế, phương pháp giám sát chi phí, chiến lược Tagging, Infrastructure as Code (IaC) và giải pháp tự động hóa việc dọn dẹp.

Đọc bài viết đầy đủ trên **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2237584380339855/)**.

![alt text](<Screenshot 2026-08-07 193410.png>)
### [Blog 3 - Tối ưu chi phí Amazon EC2 với AWS Graviton (ARM)](3-BlogsPosted/3.3-Blog3)

Bài viết trình bày cách chuyển đổi khối lượng công việc trên Amazon EC2 từ các **instance kiến trúc x86** sang **AWS Graviton (ARM)** nhằm giảm đáng kể chi phí hạ tầng đồng thời cải thiện hiệu năng. Nội dung bao gồm kết quả benchmark thực tế, các bước di chuyển, cách xây dựng Docker image đa kiến trúc (Multi-Architecture) và những thực tiễn tốt nhất khi triển khai các ứng dụng cloud-native trên nền tảng ARM.

Đọc bài viết đầy đủ trên **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233445714087055/)**.

![alt text](image.png)