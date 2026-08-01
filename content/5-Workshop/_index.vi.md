---
title: "Chương 5 - Triển khai hệ thống E-Commerce Data Pipeline"
date: 2026-07-20
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Chương 5: Triển khai hệ thống E-Commerce Data Pipeline với Apache Airflow, dbt và Amazon Redshift

## Đặt vấn đề

Trong bối cảnh các doanh nghiệp thương mại điện tử ngày càng tạo ra một lượng lớn dữ liệu giao dịch mỗi ngày, nhu cầu xây dựng một hệ thống có khả năng thu thập, lưu trữ, xử lý và cung cấp dữ liệu phục vụ phân tích một cách tự động, ổn định và có thể mở rộng trở thành một yêu cầu tất yếu. Xuất phát từ nhu cầu đó, chương này trình bày toàn bộ quá trình triển khai thực tế một hệ thống **Modern Data Stack (MDS)** hoàn chỉnh mang tên **ecom-data-pipeline**, được xây dựng nhằm mô phỏng lại quy trình xử lý dữ liệu của một nền tảng thương mại điện tử ở quy mô vừa, từ khâu thu thập dữ liệu thô cho đến khâu cung cấp các bảng dữ liệu đã qua xử lý, sẵn sàng phục vụ báo cáo và phân tích nghiệp vụ.

Bộ dữ liệu được sử dụng xuyên suốt chương này là **Brazilian E-Commerce Public Dataset by Olist**, một bộ dữ liệu thực tế được công bố công khai trên nền tảng Kaggle, bao gồm hơn 100.000 đơn hàng phát sinh trong giai đoạn 2016-2018 tại thị trường Brazil, cùng với các thông tin đi kèm về khách hàng, người bán, sản phẩm, thanh toán và đánh giá của khách hàng sau khi nhận hàng. Việc lựa chọn một bộ dữ liệu thực tế, có quy mô đủ lớn và có nhiều chiều thông tin liên quan đến nhau, giúp quá trình triển khai bám sát các bài toán thường gặp trong thực tế vận hành một hệ thống dữ liệu cho doanh nghiệp, thay vì chỉ dừng lại ở mức minh họa lý thuyết.

## Mục tiêu và phạm vi triển khai

Mục tiêu tổng quát của chương là xây dựng hoàn chỉnh một pipeline dữ liệu theo kiến trúc **ELT (Extract - Load - Transform)**, trong đó dữ liệu thô được trích xuất và nạp trực tiếp vào kho dữ liệu trước, sau đó mới thực hiện các phép biến đổi ngay trên kho dữ liệu bằng công cụ chuyên dụng. Cách tiếp cận này khác với kiến trúc ETL truyền thống (biến đổi dữ liệu trước khi nạp vào kho), và được lựa chọn vì tận dụng được năng lực tính toán song song mạnh mẽ của Amazon Redshift, đồng thời giúp toàn bộ logic biến đổi dữ liệu được quản lý tập trung, có thể kiểm thử và theo dõi phiên bản thông qua dbt.

Trong phạm vi chương này, các nội dung sau được trình bày chi tiết:

- Thiết kế tổng thể hệ thống theo kiến trúc phân tầng dữ liệu (Bronze - Silver - Gold), tương ứng với các tầng Raw, Staging và Data Mart trong dbt.
- Chuẩn bị môi trường triển khai, bao gồm cấu hình phần cứng, tài khoản đám mây AWS và bộ công cụ lập trình cần thiết.
- Triển khai tầng lưu trữ dữ liệu thô trên Amazon S3 và tầng kho dữ liệu trên Amazon Redshift Serverless, cùng với việc thiết lập đường truyền riêng (Gateway VPC Endpoint) giữa các thành phần trong VPC với Amazon S3.
- Xây dựng tầng biến đổi dữ liệu bằng dbt Core, bao gồm các model ở tầng Staging và tầng Data Mart, kèm theo cơ chế kiểm thử chất lượng dữ liệu (Data Quality Testing).
- Điều phối toàn bộ quy trình bằng Apache Airflow, đảm bảo pipeline chạy tự động theo lịch, có khả năng thử lại khi gặp lỗi và có thể theo dõi trạng thái thực thi.
- Chuẩn hóa cấu hình và quy tắc quản trị cho dự án dbt, phục vụ khả năng mở rộng và bảo trì lâu dài.
- Thực hiện dọn dẹp tài nguyên sau khi hoàn tất, nhằm tránh phát sinh chi phí ngoài dự kiến trên AWS.

Chương này tập trung vào phần hạ tầng dữ liệu (data infrastructure) và tầng biến đổi dữ liệu; các nội dung liên quan đến trực quan hóa dữ liệu (dashboard/BI) trên các bảng dữ liệu đầu ra không thuộc phạm vi trình bày của chương, mặc dù các bảng Data Mart được xây dựng trong chương này hoàn toàn có thể kết nối trực tiếp tới các công cụ BI phổ biến như Amazon QuickSight, Power BI hoặc Tableau ở giai đoạn kế tiếp của dự án.

## Bố cục chương

Nội dung chương được tổ chức thành sáu phần chính, được trình bày theo đúng trình tự đã thực hiện trong quá trình triển khai thực tế:

1. [Tổng quan hệ thống](5.1-Workshop-overview/) - Trình bày mục tiêu chi tiết, kiến trúc tổng thể và vai trò của từng thành phần công nghệ trong hệ thống.
2. [Chuẩn bị môi trường triển khai](5.2-Prerequiste/) - Mô tả cấu hình phần cứng, tài khoản AWS, phân quyền IAM và bộ công cụ được sử dụng.
3. [Triển khai tầng lưu trữ và kho dữ liệu trên AWS](5.3-S3-vpc/) - Trình bày quá trình khởi tạo Amazon S3, Amazon Redshift Serverless và thiết lập kết nối riêng qua Gateway VPC Endpoint.
4. [Triển khai tầng biến đổi dữ liệu và điều phối pipeline](5.4-S3-onprem/) - Trình bày chi tiết các model dbt ở tầng Staging và Data Mart, cấu hình Airflow DAG và kết quả truy vấn phân tích.
5. [Chuẩn hóa cấu hình và quản trị dự án dbt](5.5-Policy/) - Trình bày cách tổ chức file `dbt_project.yml` và các quy ước đặt tên, phân tầng schema.
6. [Dọn dẹp tài nguyên](5.6-Cleanup/) - Trình bày quy trình thu hồi tài nguyên AWS sau khi hoàn tất quá trình triển khai và kiểm thử.

![Airflow DAG](00-workshop-header.png)