---
title: "Blog 3: AWS S3 Bucket cho Hệ thống Chăm sóc Sức khỏe Số"
date: 2026-06-15
weight: 3
chapter: false
---

# [FCAJ2026] AWS S3 Bucket: Mảnh ghép lưu trữ Cloud-Native cho Hệ thống Chăm sóc Sức khỏe Số

## Giới thiệu

Trong quá trình phát triển **Hệ thống Chăm sóc Sức khỏe Số (Digital Healthcare System)** — một nền tảng hỗ trợ tư vấn y tế bằng AI và quản lý lịch hẹn khám bệnh — nhóm chúng tôi đã triển khai thành công các chức năng như đặt lịch khám, tư vấn bằng AI và xác thực người dùng. Tuy nhiên, một thách thức quan trọng khác nhanh chóng xuất hiện: **lưu trữ và quản lý tệp tin**.

Hệ thống liên tục tạo ra nhiều loại dữ liệu khác nhau, bao gồm:

- **Bệnh nhân:** Ảnh đại diện, báo cáo y tế và đơn thuốc ở định dạng PDF.
- **Bác sĩ & Nhân viên:** Chứng chỉ hành nghề, bằng cấp chuyên môn và các tài liệu liên quan.
- **Mô hình AI:** Các tệp trọng số (`.pt`, `.bin`) có dung lượng lên đến hàng trăm MB cùng với lịch sử hội thoại của AI.

Ở giai đoạn đầu phát triển, việc lưu các tệp tải lên trong thư mục `uploads/` của backend và đưa trực tiếp các tệp trọng số AI vào kho mã nguồn Git là cách làm đơn giản và thuận tiện.

Tuy nhiên, khi chuẩn bị triển khai hệ thống lên môi trường production, cách tiếp cận này nhanh chóng bộc lộ nhiều hạn chế:

- Kho Git trở nên quá lớn và khó quản lý.
- Máy chủ backend trở thành một máy chủ lưu trữ **stateful**.
- Các tài liệu y tế nhạy cảm đối mặt với nhiều rủi ro về bảo mật.
- Việc mở rộng hệ thống trên nhiều máy chủ trở nên phức tạp do phải đồng bộ dữ liệu.

Để giải quyết những vấn đề này, dự án đã lựa chọn **Amazon Simple Storage Service (Amazon S3)** làm giải pháp lưu trữ tập trung trên nền tảng đám mây.

---

## Ứng dụng thực tế trong dự án

Thay vì để backend lưu trữ toàn bộ các tệp do người dùng tải lên, tất cả tài nguyên dạng tệp đều được chuyển sang lưu trữ trên Amazon S3.

### Bảo vệ chứng chỉ hành nghề của bác sĩ

Chứng chỉ hành nghề và giấy phép y tế chứa nhiều thông tin cá nhân nhạy cảm.

Các tài liệu này được lưu trữ trong một **S3 Bucket riêng tư (Private S3 Bucket)**.

Khi quản trị viên cần xem xét hồ sơ của bác sĩ, backend được xây dựng bằng NestJS sẽ tạo một **Presigned URL** với thời gian hiệu lực ngắn.

Sau khi URL hết hạn, quyền truy cập sẽ tự động bị thu hồi, giúp đảm bảo các tài liệu quan trọng luôn được bảo vệ.

---

### Quản lý trọng số của mô hình AI

Các tệp trọng số của mô hình AI (`model.pt`) được tách biệt hoàn toàn khỏi mã nguồn ứng dụng.

Thay vì lưu các tệp dung lượng lớn trong Git, toàn bộ checkpoint và trọng số của mô hình được lưu trên Amazon S3.

Khi dịch vụ AI khởi động, một script Python sẽ tự động tải phiên bản trọng số mới nhất từ S3 trước khi nạp mô hình vào bộ nhớ.

Kiến trúc này giúp việc quản lý phiên bản mô hình trở nên đơn giản hơn, đồng thời giữ cho kho mã nguồn luôn gọn nhẹ.

---

### Lưu trữ báo cáo y tế và lịch sử hội thoại

Sau mỗi phiên tư vấn bằng AI, hệ thống sẽ tự động:

- Tạo báo cáo y tế ở định dạng PDF.
- Lưu lại lịch sử hội thoại giữa AI và người dùng.
- Tải cả hai tài nguyên trực tiếp lên Amazon S3.

Các tệp này có thể được người dùng được cấp quyền tải xuống hoặc lưu trữ lâu dài để phục vụ công tác kiểm toán và tra cứu sau này.

---

## Lợi ích khi sử dụng Amazon S3

Việc tích hợp Amazon S3 mang lại nhiều lợi ích quan trọng cho dự án.

### Kiến trúc Backend Stateless

Backend NestJS giờ đây chỉ tập trung xử lý logic nghiệp vụ và cung cấp API.

Do toàn bộ tệp tĩnh được lưu trữ bên ngoài, các máy chủ backend luôn ở trạng thái **stateless** và có thể mở rộng theo chiều ngang mà không cần đồng bộ các tệp tải lên giữa nhiều máy chủ.

---

### Nâng cao bảo mật dữ liệu y tế

Các tài liệu y tế nhạy cảm không bao giờ được truy cập thông qua các đường dẫn công khai.

Thay vào đó, quyền truy cập được kiểm soát thông qua:

- AWS IAM Permissions
- Private S3 Bucket
- Presigned URL có thời hạn

Nhờ đó, chỉ những người dùng được cấp quyền mới có thể truy cập các tài liệu y tế được bảo vệ.

---

### Độ tin cậy cao với chi phí tối ưu

Amazon S3 cung cấp độ bền dữ liệu lên tới **99.999999999% (11 số 9)**, thuộc hàng cao nhất trong ngành.

Bên cạnh đó, AWS còn cung cấp **Free Tier**, cho phép lưu trữ miễn phí tới **5 GB** trong 12 tháng đầu tiên.

Kết hợp với **S3 Lifecycle Rules** và các lớp lưu trữ như **Amazon S3 Glacier**, các tệp ít được truy cập hoặc nhật ký tạm thời có thể được tự động lưu trữ lâu dài, giúp tối ưu đáng kể chi phí lưu trữ.

---

## Kết luận

Việc áp dụng các dịch vụ lưu trữ cloud-native như Amazon S3 là một thực tiễn kiến trúc quan trọng đối với các ứng dụng hiện đại.

Thay vì biến máy chủ backend thành nơi lưu trữ tệp với nhiều vấn đề về đồng bộ và bảo mật, nhà phát triển có thể giao toàn bộ trách nhiệm lưu trữ cho AWS.

Cách tiếp cận này giúp xây dựng một hệ thống an toàn hơn, dễ mở rộng hơn và dễ bảo trì hơn, đồng thời cho phép đội ngũ phát triển tập trung vào việc xây dựng các tính năng mang lại giá trị cho người dùng thay vì quản lý hạ tầng lưu trữ.

---

## Tài liệu tham khảo

- AWS. **Hướng dẫn sử dụng Amazon Simple Storage Service (S3)**  
  https://docs.aws.amazon.com/AmazonS3/latest/userguide/

- AWS. **Tổng quan về Amazon S3**  
  https://aws.amazon.com/s3/