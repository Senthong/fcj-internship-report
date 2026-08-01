---
title: "5.3.1 - Khởi tạo hạ tầng lưu trữ và kết nối riêng tới Amazon S3"
weight: 1
chapter: false
pre: " <b> 5.3.1 </b> "
---

Mục này trình bày ba nhóm công việc được thực hiện tuần tự: khởi tạo Amazon S3 Bucket, khởi tạo Amazon Redshift Serverless, và cấu hình Gateway VPC Endpoint để kết nối hai thành phần này với nhau qua đường mạng riêng. Trong dự án thực tế, ba nhóm công việc này được quản lý phần lớn thông qua Terraform (`infrastructure/terraform/main.tf`); nội dung dưới đây trình bày lại các bước tương ứng khi thao tác trực tiếp qua AWS Management Console, phục vụ mục đích minh họa và đối chiếu với cấu hình mã nguồn.

## Bước 1: Khởi tạo Amazon S3 Bucket cho tầng lưu trữ dữ liệu thô

Theo quy ước đặt tên được sử dụng xuyên suốt dự án (tiền tố `ecom-pipeline`, hậu tố là tên môi trường), hai bucket được khởi tạo như sau:

1. Truy cập **Amazon S3 Console**, chọn **Create bucket**.
2. Đặt tên bucket thứ nhất là `ecom-pipeline-raw-prod`, tương ứng tầng Bronze - nơi lưu trữ nguyên vẹn dữ liệu CSV tải về từ Kaggle, được script `ingestion/ingest_olist_to_s3.py` ghi vào theo cấu trúc phân vùng `olist/year=<năm>/month=<tháng>/day=<ngày>/`.
3. Chọn khu vực (Region) `ap-southeast-2`, thống nhất với khu vực được khai báo mặc định trong biến `aws_region` của Terraform và trong script ingestion.
4. Giữ nguyên trạng thái bật của mục **Block all public access** - đây là yêu cầu bắt buộc vì bucket chứa dữ liệu giao dịch không được phép truy cập công khai dưới bất kỳ hình thức nào.
5. Bật **Bucket Versioning** ngay tại bước khởi tạo, nhằm cho phép khôi phục lại phiên bản trước đó của một tệp trong trường hợp script ingestion ghi đè nhầm dữ liệu.
6. Lặp lại các bước trên để khởi tạo bucket thứ hai, đặt tên `ecom-pipeline-processed-prod`, tương ứng tầng Silver - nơi dự kiến lưu trữ dữ liệu trung gian đã qua xử lý sơ bộ trước khi nạp vào Redshift.

Sau khi hai bucket được tạo, một cấu hình vòng đời (Lifecycle Configuration) được thiết lập cho bucket `raw`, áp dụng cho tất cả object có tiền tố `olist/`: sau 90 ngày kể từ thời điểm tạo, object được tự động chuyển sang lớp lưu trữ **Intelligent-Tiering** nhằm tối ưu chi phí lưu trữ cho dữ liệu ít được truy cập; sau 365 ngày, object sẽ tự động hết hạn và bị xóa. Cấu hình này phù hợp với đặc điểm sử dụng của dữ liệu Olist - phần lớn các truy vấn phân tích chỉ cần dữ liệu trong khoảng thời gian gần, trong khi dữ liệu phân vùng theo ngày của các lần chạy ingestion cũ hơn ít khi được truy xuất lại.

![S3 bucket](5.3.1-s3-bucket.png)

## Bước 2: Khởi tạo Amazon Redshift Serverless

Khác với việc khởi tạo một Provisioned Cluster (yêu cầu chọn loại node cố định như `dc2.large` hoặc `ra3.xlplus`), Redshift Serverless được cấu hình theo hai đối tượng riêng biệt: **Namespace** (không gian tên, gắn với dữ liệu và bảo mật) và **Workgroup** (nhóm làm việc, gắn với năng lực tính toán). Việc tách hai khái niệm này cho phép nhiều Workgroup có mức công suất khác nhau cùng truy cập một Namespace dữ liệu, dù trong phạm vi dự án hiện tại chỉ sử dụng một Workgroup duy nhất.

1. Truy cập **Amazon Redshift Console**, chọn mục **Redshift Serverless**, sau đó chọn **Create workgroup**.
2. Khai báo Namespace với tên `ecom-pipeline-ns-prod`, khai báo tên cơ sở dữ liệu mặc định là `dev` và tài khoản quản trị là `admin`, kèm theo mật khẩu đáp ứng yêu cầu độ phức tạp tối thiểu (ít nhất 8 ký tự, có chữ hoa, chữ thường và số).
3. Gắn Namespace vừa tạo với IAM Role đã được Terraform khởi tạo trước đó (`ecom-pipeline-redshift-s3-prod`), cho phép Redshift được phép đọc dữ liệu từ hai bucket S3 đã tạo ở Bước 1 mà không cần truyền Access Key/Secret Key.
4. Khai báo Workgroup với tên `ecom-pipeline-wg-prod`, mức công suất cơ sở (base capacity) là **8 RPU** (Redshift Processing Unit) - mức tối thiểu phù hợp cho môi trường thực hành và có thể điều chỉnh tăng lên khi khối lượng truy vấn tăng.
5. Ở bước cấu hình mạng, tắt tùy chọn **Turn on publicly accessible**, chọn các subnet riêng tư thuộc VPC đã chuẩn bị, và gắn Security Group riêng cho Redshift (`ecom-pipeline-redshift-sg-prod`) - Security Group này chỉ cho phép lưu lượng truy cập vào cổng `5439` xuất phát từ dải địa chỉ CIDR nội bộ của VPC, không mở cổng này ra Internet công cộng dưới bất kỳ hình thức nào.

Sau khi Workgroup được khởi tạo thành công và chuyển sang trạng thái **Available**, hai schema cần thiết cho các bước tiếp theo được tạo trực tiếp bằng câu lệnh SQL, thực thi thông qua Query Editor v2 tích hợp sẵn trong Redshift Console:

```sql
CREATE SCHEMA staging;
CREATE SCHEMA mart;
```

Trong đó, schema `staging` tương ứng với tầng Silver (nơi các bảng dữ liệu thô sau khi nạp bằng lệnh `COPY` được lưu tạm trước khi qua các phép biến đổi của dbt), còn schema `mart` tương ứng với tầng Gold (nơi lưu trữ các bảng dữ liệu đã tổng hợp, sẵn sàng phục vụ báo cáo nghiệp vụ).

![ecom-pipeline-wg-prod](image.png)

## Bước 3: Cấu hình Gateway VPC Endpoint cho Amazon S3

Đây là bước quan trọng nhất của mục 5.3.1, thiết lập đường kết nối riêng để Redshift Serverless (đã được khởi tạo ở Bước 2 với `publicly_accessible = false`) có thể truy cập được hai bucket S3 đã khởi tạo ở Bước 1 mà không cần đi qua Internet công cộng hay NAT Gateway.

1. Truy cập **VPC Console**, chọn mục **Endpoints** ở thanh điều hướng bên trái, sau đó chọn **Create endpoint**.
2. Đặt tên gợi nhớ cho Endpoint, ví dụ `ecom-pipeline-s3-gateway-endpoint`.
3. Ở mục **Service category**, chọn **AWS services**.
4. Trong ô tìm kiếm dịch vụ, nhập từ khóa `s3` và chọn dòng dịch vụ có Type là **Gateway** (phân biệt với dòng có Type là **Interface** cũng xuất hiện trong kết quả tìm kiếm) - dịch vụ này có tên dạng `com.amazonaws.ap-southeast-2.s3`. Loại Gateway được lựa chọn vì đây là loại endpoint không phát sinh chi phí theo giờ hoặc theo lưu lượng, hoạt động bằng cách bổ sung một route vào Route Table của VPC, phù hợp với hai dịch vụ duy nhất hỗ trợ loại này là Amazon S3 và Amazon DynamoDB.
5. Chọn đúng VPC nơi Redshift Serverless Workgroup đang hoạt động.
6. Ở mục **Route tables**, chọn (các) Route Table đang được gắn với những subnet riêng tư chứa Redshift Workgroup - đây là bước quyết định phạm vi các subnet nào sẽ có quyền truy cập S3 qua đường riêng này; nếu bỏ sót Route Table nào, các tài nguyên trong subnet tương ứng với Route Table đó sẽ không thể truy cập S3 qua Endpoint.
7. Ở mục **Policy**, giữ tùy chọn **Full access** ở bước khởi tạo ban đầu để đảm bảo pipeline hoạt động thông suốt trong giai đoạn kiểm thử; chính sách này sẽ được thu hẹp phạm vi cụ thể hơn ở phần 5.5 của chương, chỉ cho phép truy cập đúng hai bucket phục vụ dự án thay vì toàn bộ tài nguyên S3 của tài khoản.
8. Chọn **Create endpoint** để hoàn tất.

Sau khi Endpoint chuyển sang trạng thái **Available**, AWS tự động bổ sung một route mới vào (các) Route Table đã chọn ở bước 6, với đích đến là danh sách tiền tố IP (prefix list) của Amazon S3 trong khu vực `ap-southeast-2`, và target là chính Gateway Endpoint vừa tạo. Từ thời điểm này, mọi lưu lượng từ các subnet riêng tư hướng đến Amazon S3 sẽ tự động được định tuyến qua Endpoint thay vì tìm đường ra Internet, mà không cần thay đổi bất kỳ cấu hình nào ở phía Redshift hay ứng dụng.

