---
title: "01. AWS Glue và AWS Lambda + Polars"
date: 2026-06-29
weight: 3
chapter: false
---

# [FCAJ2026] AWS Glue (PySpark) và AWS Lambda + Polars: Khi một kiến trúc Serverless đơn giản vượt trội hơn Spark

## Giới thiệu

Trong cộng đồng **Data Engineering**, có một quan điểm khá phổ biến rằng khi cần xử lý hàng chục hoặc hàng trăm GB dữ liệu trên AWS thì gần như bắt buộc phải sử dụng **AWS Glue**, **Apache Spark** hoặc **Amazon EMR**.

Tuy nhiên, trong quá trình thiết kế lại một pipeline dữ liệu production xử lý khoảng **100 GB dữ liệu giao dịch mỗi ngày**, được lưu dưới dạng các tệp Parquet và CSV nén trên Amazon S3, nhóm chúng tôi đã thử nghiệm một kiến trúc hoàn toàn khác.

Thay vì sử dụng AWS Glue, chúng tôi xây dựng một pipeline ETL hoàn toàn serverless với:

- AWS Lambda
- Polars
- AWS Step Functions (Distributed Map)

Kết quả benchmark đã khiến cả nhóm hạ tầng thực sự bất ngờ.

---

## Kết quả Benchmark

Cả hai giải pháp đều xử lý chính xác cùng một tập dữ liệu production.

| Chỉ số | AWS Glue (8 DPUs) | AWS Lambda + Polars |
|---------|------------------:|--------------------:|
| Thời gian xử lý | ~11 phút 40 giây | ~2 phút 15 giây |
| Thời gian khởi động | 2–3 phút | < 800 ms |
| Chi phí mỗi lần chạy | ~1,68 USD | ~0,11 USD |
| Độ phức tạp của mã nguồn | Cao | Thấp |

So với AWS Glue, kiến trúc serverless đạt được:

- **Nhanh hơn khoảng 5,2 lần**
- **Giảm hơn 93% chi phí xử lý**
- **Hầu như không cần quản lý hạ tầng**

Nhờ đó, AWS Lambda + Polars đã trở thành giải pháp ETL chính trong môi trường production, trong khi AWS Glue vẫn được giữ lại cho các bài toán xử lý dữ liệu ở quy mô petabyte trong tương lai.

---

## Vì sao Lambda lại nhanh hơn?

Khác biệt lớn nhất đến từ cơ chế thực thi bên dưới.

AWS Glue được xây dựng trên Apache Spark, vì vậy mỗi lần chạy đều phải trải qua các bước như:

- Khởi tạo Spark Driver.
- Cấp phát Spark Executor.
- Khởi động JVM.
- Giao tiếp giữa Python và JVM thông qua Py4J.
- Tuần tự hóa (Serialization) và giải tuần tự hóa (Deserialization) dữ liệu.

Mặc dù Spark rất mạnh đối với các bài toán phân tán quy mô lớn, những chi phí khởi tạo này lại trở thành điểm bất lợi đối với các tập dữ liệu cỡ trung bình.

Trong khi đó, Polars được viết hoàn toàn bằng **Rust** và xây dựng trên nền **Apache Arrow**.

Bộ máy xử lý của Polars hỗ trợ:

- Xử lý đa luồng (Multi-threading).
- Thực thi theo vector (Vectorized Execution).
- Các thao tác bộ nhớ Zero-Copy.

Tất cả đều diễn ra mà không cần JVM, giúp tận dụng tài nguyên phần cứng hiệu quả hơn đáng kể.

---

## Chia để trị với AWS Step Functions

Thay vì xử lý toàn bộ 100 GB dữ liệu trong một ETL Job duy nhất, pipeline áp dụng chiến lược **Divide and Conquer**.

Dữ liệu trên Amazon S3 được phân vùng theo:

- Năm (Year)
- Tháng (Month)
- Ngày (Day)
- Giờ (Hour)

AWS Step Functions Distributed Map sẽ khởi chạy nhiều AWS Lambda song song.

Mỗi Lambda chỉ xử lý một phân vùng dữ liệu.

Ví dụ:

- 50 Lambda Function
- 10 GB bộ nhớ cho mỗi Lambda
- Tối đa 6 vCPU
- Mỗi Lambda xử lý độc lập một partition

Kiến trúc này tận dụng tối đa khả năng thực thi đồng thời (Concurrency) của AWS Lambda trong khi vẫn giữ cho mỗi lần thực thi nhẹ và hiệu quả.

---

## Loại bỏ Spark Shuffle

Một trong những điểm nghẽn hiệu năng lớn nhất của Spark là **Shuffle Operation**.

Mỗi khi Join hoặc Aggregate yêu cầu di chuyển dữ liệu giữa các Worker Node, Spark phải dành rất nhiều thời gian để truyền dữ liệu trung gian qua mạng.

Trong pipeline này, nhóm chúng tôi giảm thiểu tối đa việc truyền dữ liệu bằng cách thiết kế cấu trúc lưu trữ trên Amazon S3 một cách hợp lý.

Nhờ sử dụng **Partition Pruning** ngay từ giai đoạn đọc dữ liệu, mỗi Lambda chỉ tải đúng những partition cần thiết.

Do đó, không còn các Shuffle Operation tốn kém như trong Spark.

---

## Lợi ích không chỉ nằm ở hiệu năng

### Giảm chi phí trên nền tảng đám mây

AWS Lambda sử dụng mô hình tính phí **Pay-as-you-go** thực sự.

Đối với các tổ chức vận hành hàng trăm pipeline ETL mỗi ngày, việc giảm hơn 90% chi phí xử lý có thể giúp tiết kiệm hàng nghìn USD mỗi tháng.

---

### Không cần quản lý Cluster

Khác với Spark Cluster, kiến trúc Lambda không yêu cầu:

- Khởi tạo Cluster.
- Tinh chỉnh JVM.
- Cấu hình Executor.
- Tối ưu bộ nhớ.
- Quản lý phiên bản Spark.

Việc triển khai chỉ đơn giản là một Docker Image nhỏ gọn chứa Python và Polars.

---

### Trải nghiệm phát triển đơn giản hơn

Việc chạy PySpark trên máy cá nhân thường yêu cầu nhiều bước cấu hình và tiêu tốn khá nhiều tài nguyên.

Trong khi đó, Polars chỉ là một thư viện Python thông thường.

Lập trình viên có thể chạy, kiểm thử và gỡ lỗi ngay trên máy cá nhân với rất ít cấu hình trước khi triển khai chính Docker Image đó lên AWS Lambda.

---

## AWS Glue còn phù hợp không?

Câu trả lời là **có**.

Kết quả benchmark này **không có nghĩa** Apache Spark hay AWS Glue đã lỗi thời.

AWS Glue vẫn là lựa chọn phù hợp cho:

- Xử lý dữ liệu ở quy mô petabyte.
- Các phép Join phân tán quy mô lớn.
- Các tác vụ yêu cầu dung lượng bộ nhớ rất lớn.
- Các Data Lake doanh nghiệp có kiến trúc phức tạp.

Spark được thiết kế để giải quyết những bài toán vượt quá khả năng của một máy tính hoặc một instance đơn lẻ.

---

## Kết luận

Đối với nhiều pipeline ETL doanh nghiệp có quy mô từ vài GB đến vài TB dữ liệu, một kiến trúc serverless được xây dựng bằng **AWS Lambda**, **AWS Step Functions** và các công cụ xử lý dữ liệu hiện đại như **Polars** có thể mang lại tỷ lệ **hiệu năng trên chi phí (Performance-to-Cost)** vượt trội hơn đáng kể so với các giải pháp dựa trên Spark truyền thống.

Thay vì mặc định triển khai một cụm xử lý phân tán lớn, các kỹ sư nên bắt đầu bằng một câu hỏi đơn giản:

> **Liệu bài toán này có thể được giải quyết hiệu quả hơn bằng một kiến trúc serverless gọn nhẹ hay không?**

Trong rất nhiều trường hợp thực tế, câu trả lời là **có**.

---

## Tài liệu tham khảo

Đọc bài viết đầy đủ trên **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233435944088032/)**.

- AWS. **AWS Lambda Developer Guide**  
  https://docs.aws.amazon.com/lambda/

- Polars. **Blazingly Fast DataFrames Library**  
  https://pola.rs/

- AWS. **AWS Step Functions Distributed Map**  
  https://aws.amazon.com/step-functions/

- Apache Arrow. **Architecture Documentation**  
  https://arrow.apache.org/