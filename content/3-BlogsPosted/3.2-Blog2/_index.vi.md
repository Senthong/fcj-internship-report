---
title: "Blog 2: AWS Cognito cho Hệ thống Chăm sóc Sức khỏe Số"
date: 2026-06-08
weight: 2
chapter: false
---

# [FCAJ2026] AWS Cognito là gì? Vì sao đây là mảnh ghép không thể thiếu của Hệ thống Chăm sóc Sức khỏe Số?

## Giới thiệu

Trong quá trình phát triển **Nền tảng Chăm sóc Sức khỏe Thông minh (Smart Healthcare Platform)** — một hệ thống hỗ trợ phân loại triệu chứng bằng AI và đặt lịch khám trực tuyến — việc quản lý danh tính người dùng và kiểm soát quyền truy cập theo nhiều vai trò là một trong những thách thức lớn nhất về mặt kiến trúc.

Nền tảng hỗ trợ ba nhóm người dùng hoàn toàn khác nhau:

- Bệnh nhân
- Bác sĩ
- Nhân viên lễ tân

Mỗi vai trò đều có quyền hạn và mức độ truy cập riêng. Thay vì tự xây dựng và duy trì một hệ thống xác thực người dùng, dự án đã tích hợp **AWS Cognito** như một dịch vụ quản lý danh tính được AWS vận hành hoàn toàn.

Quyết định này giúp đơn giản hóa quá trình xác thực, phân quyền và quản lý người dùng, đồng thời nâng cao tính bảo mật cho toàn bộ hệ thống chăm sóc sức khỏe.

---

## AWS Cognito là gì?

AWS Cognito là một dịch vụ **quản lý danh tính và quyền truy cập (Identity and Access Management) được AWS quản lý hoàn toàn**, cho phép nhà phát triển xác thực và phân quyền người dùng mà không cần tự xây dựng hạ tầng xác thực từ đầu.

Dịch vụ cung cấp hai thành phần cốt lõi:

- **User Pools** để quản lý tài khoản người dùng và quá trình xác thực.
- **Identity Pools** để cấp quyền truy cập an toàn tới các tài nguyên AWS cho người dùng đã được xác thực.

Nhờ sử dụng AWS Cognito, nhóm phát triển có thể tập trung xây dựng các tính năng của ứng dụng, trong khi AWS đảm nhiệm việc quản lý mật khẩu, tạo token, xác minh người dùng và áp dụng các thực tiễn bảo mật tốt nhất.

---

## Giải quyết bài toán phân quyền đa vai trò

Một trong những yêu cầu quan trọng nhất của Smart Healthcare Platform là hỗ trợ nhiều nhóm người dùng với quyền truy cập khác nhau.

Hệ thống triển khai mô hình **Role-Based Access Control (RBAC)** thông qua **Cognito Groups**.

Mỗi người dùng sẽ thuộc về một nhóm cụ thể, chẳng hạn như:

- Patient
- Doctor
- Receptionist

Sau khi người dùng đăng nhập thành công, AWS Cognito sẽ phát hành một JWT Token chứa thuộc tính **`cognito:groups`**.

Backend sẽ xác thực token và xác định quyền của người dùng dựa trực tiếp trên thông tin nhóm được nhúng trong token, giúp loại bỏ nhu cầu xây dựng các cơ chế phân quyền phức tạp trong ứng dụng.

---

## Quy trình tạo tài khoản an toàn cho nhân viên y tế

Việc quản lý tài khoản của bác sĩ và nhân viên lễ tân đòi hỏi một quy trình khởi tạo tài khoản an toàn.

Thay vì phải tự đặt mật khẩu cho từng người dùng, quản trị viên chỉ cần tạo tài khoản thông qua API của backend.

AWS Cognito sẽ tự động:

- Tạo tài khoản người dùng.
- Gửi email chứa mật khẩu tạm thời.
- Yêu cầu người dùng đổi mật khẩu trong lần đăng nhập đầu tiên.

Quy trình này vừa nâng cao tính bảo mật vừa giảm đáng kể khối lượng công việc quản trị.

---

## Xác thực JWT Token an toàn

Dịch vụ backend được xây dựng bằng **NestJS** không lưu mật khẩu của người dùng trong cơ sở dữ liệu.

Thay vào đó, mọi JWT Token gửi đến đều được xác thực bằng **Public Keys** do AWS Cognito cung cấp.

Cách tiếp cận này mang lại nhiều lợi ích:

- Mật khẩu không bao giờ tồn tại trong cơ sở dữ liệu của backend.
- Quá trình xác thực tuân thủ các tiêu chuẩn bảo mật phổ biến.
- Việc xác minh token đơn giản, đáng tin cậy và có khả năng mở rộng cao.

---

## Bảo vệ dữ liệu y tế nhạy cảm

Các hệ thống chăm sóc sức khỏe phải đáp ứng những yêu cầu nghiêm ngặt về quyền riêng tư và bảo mật dữ liệu.

Để tăng cường bảo vệ danh tính người dùng, nền tảng còn áp dụng **chiến lược ẩn danh hóa bằng UUID**, giúp tránh việc lộ các mã định danh nội bộ của hệ thống.

Kết hợp với cơ chế xác thực của AWS Cognito, kiến trúc này bổ sung thêm một lớp bảo vệ cho các dữ liệu y tế nhạy cảm.

---

## Tối ưu chi phí

Một ưu điểm nổi bật khác của AWS Cognito là khả năng tối ưu chi phí.

Dịch vụ cung cấp **AWS Free Tier** với khả năng hỗ trợ tới **50.000 Monthly Active Users (MAUs)**, phù hợp với các startup, dự án học tập và các ứng dụng chăm sóc sức khỏe trong giai đoạn đầu.

Do AWS chịu trách nhiệm vận hành toàn bộ hạ tầng xác thực, nhóm phát triển không cần triển khai hay bảo trì các máy chủ xác thực riêng, từ đó giảm cả chi phí vận hành lẫn chi phí hạ tầng.

---

## Kết luận

AWS Cognito không chỉ đơn thuần là một dịch vụ đăng nhập.

Đây là một nền tảng quản lý danh tính được AWS quản lý hoàn toàn, giúp đơn giản hóa việc xác thực, phân quyền, quản lý người dùng và tăng cường bảo mật cho các ứng dụng cloud-native hiện đại.

Đối với các hệ thống chăm sóc sức khỏe số, việc tích hợp AWS Cognito giúp nhóm phát triển giảm bớt sự phức tạp của hạ tầng xác thực để tập trung xây dựng những tính năng mang lại giá trị thực sự cho bệnh nhân và quy trình khám chữa bệnh.

---

## Đọc bài viết đầy đủ

Xem chi tiết **[tại đây](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2228021697962790/?rdid=MLFJXbxXfs8sMMKH)**.