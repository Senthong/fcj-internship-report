---
title: "Blog 3: Tối ưu chi phí AWS EC2 với AWS Graviton"
date: 2026-07-06
weight: 6
chapter: false
---

# [FCAJ2026] Tối ưu chi phí AWS EC2 bằng cách chuyển sang AWS Graviton (ARM)

## Giới thiệu

Khi xây dựng hạ tầng trên nền tảng đám mây, việc tối ưu chi phí vận hành cũng quan trọng không kém việc cải thiện hiệu năng của ứng dụng.

Trong quá trình đánh giá hạ tầng cho một môi trường production và testing quy mô nhỏ, nhóm chúng tôi đã tìm hiểu liệu việc chuyển đổi từ các **Amazon EC2 Instance sử dụng kiến trúc x86** sang **AWS Graviton (ARM)** có thể mang lại mức tiết kiệm chi phí đáng kể mà vẫn duy trì hiệu năng hay không.

Sau khi thực hiện benchmark trên nhiều microservice được đóng gói bằng Docker và xây dựng với **Node.js** và **Go**, kết quả cho thấy AWS Graviton không chỉ giúp giảm chi phí hạ tầng mà còn cải thiện hiệu quả sử dụng tài nguyên.

---

## Kết quả Benchmark

Thử nghiệm được thực hiện trên hai EC2 Instance chạy nhiều microservice dạng container.

### Kịch bản chuyển đổi

- Kiến trúc ban đầu: **t3.medium (x86)**
- Kiến trúc mới: **t4g.medium (AWS Graviton2 ARM)**

Kết quả so sánh như sau:

| Chỉ số | t3.medium | t4g.medium |
|--------|----------:|-----------:|
| Giá On-Demand | $0.0416/giờ | $0.0336/giờ |
| Chi phí EC2 hàng tháng | ~60 USD | ~48 USD |
| Mức sử dụng CPU trung bình | ~45% | ~30% |

Việc chuyển đổi mang lại:

- **Giảm khoảng 19% chi phí EC2 theo giờ**
- **Giảm khoảng 33% mức sử dụng CPU**
- Hiệu quả sử dụng tài nguyên tốt hơn cho cùng một khối lượng công việc

---

## Cơ hội Right-Sizing

Việc CPU được sử dụng ít hơn mở ra cơ hội tiếp tục tối ưu hạ tầng.

Thay vì vận hành:

- 2 × t3.medium

Hệ thống vẫn hoạt động ổn định với:

- 1 × t4g.medium
- 1 × t4g.small

Việc **Right-Sizing** này tiếp tục giúp giảm chi phí hạ tầng hàng tháng.

| Kiến trúc | Chi phí hàng tháng |
|------------|------------------:|
| Ban đầu | ~60 USD |
| Sau tối ưu | ~42 USD |

Tổng cộng, việc chuyển đổi giúp giảm khoảng **30% chi phí EC2**, tương đương tiết kiệm khoảng **18 USD mỗi tháng** đối với môi trường thử nghiệm nhỏ.

Mặc dù con số này có vẻ không lớn, nhưng khi áp dụng cho hàng chục hoặc hàng trăm EC2 Instance, khoản tiết kiệm sẽ trở nên rất đáng kể.

---

## Quy trình chuyển đổi

### Xây dựng Docker Image đa kiến trúc

Pipeline CI/CD được cập nhật bằng **GitHub Actions** kết hợp với Docker Buildx.

Docker Image hiện được build đồng thời cho cả hai nền tảng:

- `linux/amd64`
- `linux/arm64`

Điều này cho phép cùng một pipeline triển khai trên cả Intel/AMD và AWS Graviton.

---

### Kiểm tra khả năng tương thích của ứng dụng

Trước khi chuyển sang môi trường production, toàn bộ thư viện và dependency của ứng dụng được kiểm thử trên kiến trúc ARM64.

May mắn là hầu hết các runtime hiện đại đều đã hỗ trợ ARM rất tốt, bao gồm:

- Node.js
- Go
- Python
- Java

Trong trường hợp của nhóm chúng tôi, không cần thay đổi bất kỳ dòng mã nguồn nào.

---

### Triển khai hạ tầng mới

Các tệp định nghĩa hạ tầng bằng Terraform hoặc CloudFormation được cập nhật để khởi tạo AWS Graviton Instance.

Lưu lượng truy cập được chuyển dần từ các EC2 Instance sử dụng x86 sang các Instance ARM trước khi hạ tầng cũ được ngừng hoạt động hoàn toàn.

---

## Vì sao AWS Graviton có hiệu năng tốt hơn?

Bộ xử lý AWS Graviton được thiết kế chuyên biệt cho các ứng dụng cloud-native.

So với các EC2 Instance sử dụng kiến trúc x86 tương đương, AWS Graviton thường mang lại:

- Chi phí theo giờ thấp hơn.
- Tỷ lệ hiệu năng trên chi phí (Price-to-Performance) tốt hơn.
- Hiệu quả sử dụng năng lượng cao hơn.
- Hiệu năng rất tốt đối với các dịch vụ web và ứng dụng container.

Đối với phần lớn các API và backend service, AWS Graviton cung cấp đủ năng lực xử lý trong khi tiêu thụ ít tài nguyên hạ tầng hơn.

---

## Bài học rút ra

Quá trình chuyển đổi này cho thấy hai chiến lược tối ưu quan trọng.

### Giảm chi phí hạ tầng

Chỉ cần chuyển từ các EC2 Instance sử dụng x86 sang AWS Graviton đã giúp giảm gần **20% chi phí EC2 theo giờ** mà không cần thay đổi chức năng của ứng dụng.

---

### Nâng cao hiệu quả sử dụng tài nguyên

Do mức sử dụng CPU giảm đáng kể sau khi chuyển đổi, hạ tầng có thể được **Right-Sizing** xuống các loại Instance nhỏ hơn, từ đó tiếp tục tiết kiệm chi phí trong dài hạn.

Thay vì xem việc lựa chọn EC2 Instance là quyết định chỉ thực hiện một lần, việc thường xuyên đánh giá mức sử dụng tài nguyên sẽ giúp phát hiện nhiều cơ hội tối ưu chi phí.

---

## Kết luận

Chuyển đổi khối lượng công việc sang AWS Graviton là một trong những cách đơn giản và hiệu quả nhất để giảm chi phí hạ tầng AWS trong khi vẫn duy trì, thậm chí cải thiện hiệu năng của ứng dụng.

Đối với các ứng dụng container hiện đại được xây dựng bằng Node.js, Go, Python hoặc Java, quá trình chuyển đổi thường rất thuận lợi nhờ sự hỗ trợ rộng rãi của kiến trúc ARM64.

Kết hợp với chiến lược **Right-Sizing** và Docker Image đa kiến trúc, AWS Graviton giúp các tổ chức xây dựng hạ tầng cloud-native hiệu quả hơn, tiết kiệm hơn và sẵn sàng mở rộng trong tương lai.

---

## Tài liệu tham khảo

Đọc bài viết đầy đủ trên **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233445714087055/)**.

- AWS. **AWS Graviton Processors**  
  https://aws.amazon.com/ec2/graviton/

- AWS. **Amazon EC2 On-Demand Pricing**  
  https://aws.amazon.com/ec2/pricing/on-demand/

- AWS. **AWS Graviton Getting Started Guide**  
  https://github.com/aws/aws-graviton-getting-started

- Docker. **Multi-platform Builds**  
  https://docs.docker.com/build/building/multi-platform/