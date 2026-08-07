---
title: "03. Truy tìm và tiêu diệt các tài nguyên Zombies trên AWS"
date: 2026-07-07
weight: 4
chapter: false
---

# [FCAJ2026] Truy tìm và tiêu diệt các tài nguyên "Zombies" trên AWS – Bài học tối ưu chi phí thực tế

## Giới thiệu

Trong quá trình vận chuyển ứng dụng và quản trị hạ tầng trên AWS, có một kịch bản rất quen thuộc mà hầu như Cloud Engineer hay DevOps nào cũng từng trải qua:

> **Hóa đơn AWS cuối tháng tăng vọt bất thường, dù bạn chắc chắn rằng các máy chủ thử nghiệm (EC2) đã được tắt từ tuần trước.**

Khi kiểm tra chi tiết trong **AWS Cost Explorer**, bạn ngỡ ngàng phát hiện ra hàng loạt khoản phí lắt nhắt đến từ những thành phần "không ai dùng đến" nhưng vẫn âm thầm chạy và tính tiền 24/7. Trong cộng đồng Cloud & FinOps, những tài nguyên này được gọi bằng một cái tên rất trực quan: **Tài nguyên "Zombies" (Zombie Resources)**.

Bài viết này sẽ giúp bạn hiểu rõ bản chất của các tài nguyên Zombie, nhận diện những "thủ phạm" phổ biến nhất gây thất thoát ngân sách, cùng quy trình từng bước để truy tìm, tiêu diệt và phòng ngừa chúng tái diễn trên hệ thống của bạn.

---

## Vì sao tài nguyên Zombies lại âm thầm ngốn tiền?

Một trong những ưu điểm lớn nhất của AWS là khả năng khởi tạo tài nguyên linh hoạt chỉ với vài cú click chuột hoặc một dòng lệnh CLI. Tuy nhiên, tính linh hoạt này cũng là con dao hai lưỡi.

Khi phát triển hoặc thử nghiệm ứng dụng (POC/Staging), kĩ sư thường tạo nhanh các EC2 Instance, đi kèm với ổ cứng EBS, địa chỉ Elastic IP, Load Balancer, NAT Gateway hay các bản Snapshot sao lưu. Khi thử nghiệm xong, chúng ta thường có thói quen bấm **Stop** hoặc **Terminate** máy chủ EC2.

**Tuy nhiên, cơ chế tính tiền của AWS lại chia nhỏ theo từng dịch vụ:**

1. **Compute (Máy tính):** Bị dừng tính tiền khi EC2 ở trạng thái `Stopped`.
2. **Storage & Networking (Lưu trữ & Mạng):** **Vẫn tiếp tục tính tiền** chừng nào tài nguyên đó còn tồn tại trong tài khoản của bạn, bất kể nó có được gắn vào EC2 hay không!

Một lầm tưởng phổ biến là *"Tắt EC2 nghĩa là dừng toàn bộ chi phí"*. Thực tế, các tài nguyên mồ côi (orphaned resources) nằm lại đằng sau vẫn hoạt động như những "bóng ma", âm thầm rút tiền từ thẻ tín dụng của bạn hàng ngày.

---

## Top 4 "Thủ phạm Zombie" phổ biến nhất trên AWS

### 1. Unattached EBS Volumes (Ổ cứng mồ côi)

Khi tạo một EC2 Instance, AWS sẽ tạo kèm các ổ đĩa **Amazon EBS** (Elastic Block Store). Nếu khi xóa EC2, bạn không tích chọn tùy chọn `Delete on Termination` cho các ổ đĩa phụ (Additional Volumes), các ổ EBS này sẽ chuyển sang trạng thái `available` (unattached).

- **Mức độ lãng phí:** EBS bị tính tiền theo dung lượng lưu trữ được cấp phát (provisioned capacity) tính theo GB/tháng, không quan tâm bạn có ghi dữ liệu hay không.
- **Ví dụ:** Một ổ `gp3` dung lượng 500 GB bỏ quên có thể ngốn của bạn ~$40 - $50 USD/tháng mà không đem lại bất kỳ giá trị nào.

---

### 2. Unassociated Elastic IP Addresses (Địa chỉ IP tĩnh không sử dụng)

AWS cung cấp địa chỉ IP tĩnh công cộng (**Elastic IP - EIP**) miễn phí khi bạn gắn (associate) nó vào một EC2 Instance đang chạy. Tuy nhiên, nếu bạn:
- Tắt EC2 Instance (`Stopped`).
- Hoặc gỡ IP đó ra khỏi EC2 nhưng không giải phóng (release) về pool của AWS.

AWS sẽ bắt đầu tính phí phạt khoảng **$0.005 USD/giờ** (~$3.60 USD/tháng) cho mỗi EIP nằm không. Lý do là AWS muốn khuyến khích người dùng trả lại tài nguyên IPv4 vốn đang ngày càng khan hiếm trên toàn cầu.

---

### 3. Old & Orphaned Snapshots / AMIs (Bản sao lưu cũ)

Trong quá trình bảo trì hoặc nâng cấp, chúng ta thường tạo các bản sao lưu **EBS Snapshot** hoặc **Amazon Machine Images (AMI)**. 

Qua thời gian, ứng dụng đã nâng cấp lên nhiều phiên bản mới, nhưng hàng chục bản Snapshot dung lượng hàng trăm GB từ nhiều tháng hay nhiều năm trước vẫn nằm nguyên đó. Do Snapshot lưu trữ trên S3 dưới dạng delta, chi phí tích lũy hàng tháng cho các bản backup cũ này có thể lên tới hàng trăm đô-la nếu không có chính sách dọn dẹp.

---

### 4. Idle Load Balancers & Unused NAT Gateways

- **NAT Gateway:** Mỗi NAT Gateway được khởi tạo trong một VPC sẽ tốn khoảng **$0.045 USD/giờ** (~$32 USD/tháng) chỉ riêng tiền phí duy trì sẵn có (base hourly rate), chưa tính phí truyền tải dữ liệu (data processing). Nếu bạn dựng NAT Gateway cho môi trường Dev/Test rồi bỏ quên, nó sẽ ngốn ít nhất $32 USD mỗi tháng.
- **Application / Network Load Balancer (ALB/NLB):** Một Load Balancer không có target group nào active hoặc không nhận traffic vẫn thu phí duy trì LCU (Load Balancer Capacity Units) tối thiểu.

---

## Quy trình 4 bước truy tìm và tiêu diệt Zombies

### Bước 1 – Rà soát thủ công bằng AWS CLI

Bạn có thể sử dụng các câu lệnh AWS CLI đơn giản dưới đây để nhanh chóng liệt kê các tài nguyên Zombie trong Region hiện tại:

**Tìm các ổ đĩa EBS mồ côi (status = available):**
```bash
aws ec2 describe-volumes     --filters Name=status,Values=available     --query "Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}"     --output table
```

**Tìm các Elastic IP không gắn với tài nguyên nào:**
```bash
aws ec2 describe-addresses     --query "Addresses[?AssociationId==null].{IP:PublicIp,AllocationId:AllocId}"     --output table
```

**Tìm các Elastic Network Interface (ENI) chưa gắn:**
```bash
aws ec2 describe-network-interfaces     --filters Name=status,Values=available     --query "NetworkInterfaces[*].{ID:NetworkInterfaceId,Vpc:VpcId,Subnet:SubnetId}"     --output table
```

---

### Bước 2 – Tận dụng công cụ AWS Trusted Advisor & Cost Explorer

Nếu bạn muốn có cái nhìn tổng quan trên toàn bộ AWS Account:

1. **AWS Trusted Advisor:** Truy cập dashboard của Trusted Advisor và kiểm tra mục **Cost Optimization**. Hệ thống sẽ tự động quét và đánh giá các mục:
   - *Unattached EBS Volumes*
   - *Unassociated Elastic IP Addresses*
   - *Idle Load Balancers*
   - *Low Utilization Amazon EC2 Instances*
2. **AWS Cost Anomaly Detection:** Bật tính năng cảnh báo bất thường về chi phí. Nếu có một tài nguyên Zombie mới phát sinh làm tăng đột biến chi phí trong ngày, AWS sẽ gửi cảnh báo ngay qua Email hoặc Slack/Teams.

---

### Bước 3 – Gắn nhãn (Tagging) và áp dụng văn hóa IaC

Nguyên nhân cốt lõi sinh ra Zombies là thiếu tính trách nhiệm và minh bạch đối với tài nguyên.

- **Thiết lập Tagging Strategy bắt buộc:** Mọi tài nguyên khi tạo ra đều phải gắn các Tag chuẩn như:
  - `Environment` (Dev / Staging / Prod)
  - `Owner` (email/tên người tạo)
  - `ExpirationDate` (Ngày dự kiến tiêu hủy cho môi trường Test)
- **Triển khai Infrastructure as Code (IaC):** Sử dụng **Terraform** hoặc **AWS CloudFormation** để quản lý hạ tầng. Khi cần kết thúc dự án thử nghiệm, chỉ cần chạy lệnh `terraform destroy`, toàn bộ các tài nguyên liên quan (EC2, EBS, EIP, SG, NAT) sẽ được xóa sạch sẽ, không để lại bất kỳ "vết tích Zombie" nào.

---

### Bước 4 – Tự động hóa việc dọn dẹp bằng AWS Lambda & DLM

Đối với các hệ thống quy mô lớn, việc rà soát bằng tay là không khả thi. Hãy áp dụng tự động hóa:

1. **Quản lý Snapshot tự động:** Sử dụng **Amazon Data Lifecycle Manager (DLM)** để thiết lập chính sách tự động tạo và xóa các EBS Snapshot theo lịch trình (ví dụ: chỉ giữ bản snapshot trong 7 ngày, quá 7 ngày tự động xóa).
2. **Auto-Cleanup Script với Lambda & EventBridge:** Viết một hàm **AWS Lambda** (Python/Boto3) chạy định kỳ mỗi tuần thông qua **Amazon EventBridge**. Hàm này sẽ quét các EBS Volume ở trạng thái `available` quá 3 ngày hoặc EIP không dùng và gửi thông báo cảnh báo qua Telegram/Slack trước khi tự động xóa.

---

## Bài học rút ra

Qua quá trình quản lý và tối ưu hóa chi phí cloud cho nhiều dự án, mình rút ra được các bài học quan trọng:

1. **Tắt không có nghĩa là dừng tính tiền:** Luôn nhớ rằng dịch vụ Cloud phân tách rõ ràng giữa chi phí Tính toán (Compute) và Lưu trữ/Kết nối (Storage/Network).
2. **Dọn dẹp triệt để khi xóa máy chủ:** Khi xóa EC2 trên Console, hãy luôn kiểm tra lại xem các ổ EBS đi kèm đã được tích chọn `Delete on termination` chưa, và xóa thủ công các Elastic IP/Snapshot không còn dùng.
3. **FinOps là trách nhiệm của mọi người:** Tối ưu chi phí không chỉ là việc của bộ phận kế toán hay quản lý. Mỗi lập trình viên và kỹ sư Cloud đều cần nâng cao ý thức về chi phí (Cost Awareness) ngay từ bước viết code hay thiết kế hạ tầng.
4. **Tự động hóa là chìa khóa:** Con người luôn có thể quên, nhưng các kịch bản tự động (Automation Scripts) và chính sách Lifecycle thì không.

---

## Kết luận

Chi phí trên AWS giống như một vòi nước đang chảy: nếu bạn không kiểm soát chặt chẽ từng van nhỏ, những giọt nước thất thoát từ tài nguyên **Zombies** sẽ tích tụ thành một khoản tiền khổng lồ vào cuối tháng.

Bằng việc nắm rõ danh sách các tài nguyên mồ côi phổ biến, kết hợp giữa việc rà soát định kỳ và tự động hóa quy trình dọn dẹp, bạn không chỉ tiết kiệm được từ **20% đến 40% chi phí AWS** hàng tháng mà còn giúp hạ tầng của mình trở nên gọn gàng, an toàn và dễ quản lý hơn rất nhiều.

Hãy mở ngay trang **AWS Management Console** của bạn hôm nay, chạy các câu lệnh kiểm tra ở trên và tiến hành "tiêu diệt" những Zombie đầu tiên nhé!

---

## Tài liệu tham khảo

Đọc bài viết đầy đủ trên **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2237584380339855)**.

- AWS. **Optimizing AWS Cost with AWS Trusted Advisor**  
  https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor.html

- AWS. **Amazon EC2 Pricing & Elastic IP Costs**  
  https://aws.amazon.com/ec2/pricing/

- AWS. **Automating EBS Snapshot Lifecycle Management**  
  https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snapshot-lifecycle.html