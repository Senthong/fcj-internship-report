---
title: "Blog 2: Khắc phục lỗi AWS S3 Access Denied (403)"
date: 2026-06-22
weight: 4
chapter: false
---

# [FCAJ2026] Khắc phục lỗi AWS S3 Access Denied (403) trên EC2 – Những bài học thực tế

## Giới thiệu

Trong quá trình triển khai ứng dụng cloud-native trên AWS, mình đã gặp phải một trong những lỗi phổ biến nhưng cũng gây khó chịu nhất khi tích hợp **Amazon S3** với **Amazon EC2**:

> **AccessDenied (403 Forbidden)**

Ứng dụng hoạt động hoàn toàn bình thường trong môi trường phát triển cục bộ (local), nhưng ngay sau khi triển khai lên EC2 thì mọi thao tác với Amazon S3 đều thất bại.

Ban đầu, mình cho rằng nguyên nhân chỉ đơn giản là thiếu quyền truy cập. Tuy nhiên, sau nhiều giờ kiểm tra và gỡ lỗi, mình nhận ra rằng cơ chế phân quyền của Amazon S3 bao gồm nhiều lớp bảo mật khác nhau, chứ không chỉ phụ thuộc vào một IAM Policy.

Bài viết này tổng hợp nguyên nhân gốc rễ, quá trình xử lý và những bài học mình rút ra khi giải quyết lỗi này.

---

## Vì sao chạy được trên Local nhưng lại lỗi trên EC2?

Trong quá trình phát triển, mình xác thực thông qua một **IAM User** được cấu hình bằng AWS CLI.

IAM User này được gán sẵn chính sách **AmazonS3FullAccess**, vì vậy mọi thao tác với Amazon S3 đều thành công.

Khi triển khai lên môi trường production, mình làm theo khuyến nghị của AWS và chuyển sang sử dụng **IAM Role (Instance Profile)** gắn trực tiếp với EC2 Instance.

Ngay sau khi triển khai, mọi yêu cầu tới Amazon S3 đều trả về lỗi:

```text
AccessDenied (403 Forbidden)
```

Sau khi phân tích, mình xác định được hai nguyên nhân chính.

---

## Nguyên nhân 1 – Sai Resource ARN

Một trong những lỗi phổ biến nhất là sử dụng sai phạm vi của **Resource ARN** trong IAM Policy.

Ví dụ:

- Các quyền áp dụng ở cấp Bucket như `s3:ListBucket` yêu cầu ARN:

```text
arn:aws:s3:::my-bucket
```

- Trong khi các quyền áp dụng ở cấp Object như:

  - `s3:GetObject`
  - `s3:PutObject`
  - `s3:DeleteObject`

phải sử dụng ARN:

```text
arn:aws:s3:::my-bucket/*
```

Nếu thiếu phần `/*`, Amazon S3 sẽ ngay lập tức từ chối các thao tác với object và trả về lỗi **Access Denied**.

---

## Nguyên nhân 2 – Mã hóa SSE-KMS

Một nguyên nhân khác khó nhận biết hơn là khi bật **Server-Side Encryption với AWS KMS (SSE-KMS)**.

Mặc dù IAM Role đã có quyền truy cập vào S3 Bucket, nhưng EC2 Instance **không được phép sử dụng KMS Key**.

Kết quả là:

- Amazon S3 vẫn tìm thấy object.
- Nhưng AWS KMS từ chối giải mã.
- Phản hồi cuối cùng vẫn là **403 Access Denied**.

Để khắc phục, IAM Role của EC2 cần được thêm vào **KMS Key Policy** với các quyền như:

- `kms:Decrypt`
- `kms:GenerateDataKey`

---

## Các bước khắc phục sự cố

### Bước 1 – Kiểm tra IAM Role của EC2

Phân tách rõ quyền ở cấp Bucket và cấp Object.

**Bucket-level permissions:**

- `s3:ListBucket`

**Object-level permissions:**

- `s3:GetObject`
- `s3:PutObject`
- `s3:DeleteObject`

Đồng thời xác minh rằng mỗi quyền đều tham chiếu đúng Resource ARN.

---

### Bước 2 – Kiểm tra KMS Key Policy

Nếu Bucket sử dụng mã hóa SSE-KMS, hãy đảm bảo IAM Role của EC2 được cấp quyền rõ ràng để:

- Giải mã dữ liệu (`Decrypt`)
- Tạo Data Key (`GenerateDataKey`)

Nếu thiếu các quyền này, Amazon S3 sẽ không thể trả về object đã được mã hóa, ngay cả khi Bucket Policy đã được cấu hình chính xác.

---

### Bước 3 – Kiểm tra các lớp bảo mật khác

Cơ chế phân quyền của Amazon S3 bao gồm nhiều lớp đánh giá chính sách.

Khi xử lý lỗi **Access Denied**, hãy lần lượt kiểm tra:

- Bucket Policy
- IAM Policy
- KMS Key Policy
- Object Ownership
- Block Public Access
- VPC Endpoint Policy (nếu sử dụng)

Chỉ cần một **Explicit Deny** ở bất kỳ lớp nào cũng sẽ ghi đè toàn bộ các quyền **Allow**.

---

### Bước 4 – Thiết lập giám sát

Để giảm thời gian xử lý các sự cố tương tự trong tương lai, mình đã cấu hình **Amazon CloudWatch** để theo dõi các lỗi phân quyền.

Một **Metric Filter** sẽ tìm kiếm các sự kiện chứa từ khóa **AccessDenied** và gửi cảnh báo thông qua **Amazon SNS**.

Nhờ đó, các lỗi về quyền truy cập có thể được phát hiện ngay lập tức mà không cần chờ người dùng báo lỗi.

---

## Bài học rút ra

Quá trình gỡ lỗi này giúp mình hiểu rõ hơn nhiều khái niệm quan trọng về bảo mật trên AWS.

- Một ứng dụng chạy thành công trên môi trường local không đồng nghĩa với việc sẽ hoạt động trên EC2, bởi IAM User và IAM Role có cơ chế phân quyền khác nhau.

- Lỗi **AccessDenied** không phải lúc nào cũng xuất phát từ IAM Policy. Bucket Policy, KMS Key Policy, Block Public Access, Object Ownership và VPC Endpoint Policy đều có thể ảnh hưởng đến kết quả phân quyền.

- Trước khi triển khai ứng dụng, nên kiểm tra quyền bằng **IAM Policy Simulator** hoặc **IAM Access Analyzer**.

- Nếu Amazon S3 sử dụng **SSE-KMS**, hãy luôn kiểm tra cả **KMS Key Policy** bên cạnh các quyền truy cập S3.

---

## Kết luận

Mặc dù lỗi **S3 Access Denied (403)** có vẻ đơn giản, nhưng việc xử lý thường đòi hỏi phải hiểu rõ cách các dịch vụ bảo mật của AWS phối hợp với nhau.

Thông qua việc rà soát IAM Role, Resource ARN, Bucket Policy, quyền của KMS và hệ thống giám sát, mình đã giải quyết hoàn toàn sự cố này, đồng thời nâng cao mức độ bảo mật cho kiến trúc lưu trữ của ứng dụng.

Hiểu rõ các lớp phân quyền này không chỉ giúp xử lý nhanh các sự cố khi triển khai mà còn góp phần xây dựng những ứng dụng cloud-native an toàn, ổn định và đáng tin cậy hơn.

---

## Tài liệu tham khảo

Đọc bài viết đầy đủ trên **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233435944088032/)**.

- AWS. **Troubleshoot Amazon S3 403 Access Denied Errors**  
  https://repost.aws/knowledge-center/s3-troubleshoot-403

- AWS. **IAM Roles for Amazon EC2**  
  https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html

- AWS. **IAM Policy Simulator**  
  https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html