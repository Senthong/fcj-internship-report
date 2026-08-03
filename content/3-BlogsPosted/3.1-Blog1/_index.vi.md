---
title: "Blog 1: Serverless RAG với AWS Bedrock Knowledge Bases"
date: 2026-06-01
weight: 1
chapter: false
---

# [FCAJ2026] AWS Bedrock Knowledge Bases là gì? Vì sao đây là "mảnh ghép" hoàn hảo cho kiến trúc Serverless RAG?

## Giới thiệu

Sau khi tìm hiểu cách xây dựng hệ thống **Medical AI Assistant** theo phương pháp **Retrieval-Augmented Generation (RAG)** từ đầu, mình bắt đầu gặp hàng loạt thách thức trong việc tự quản lý toàn bộ pipeline dữ liệu. Điều khiến mình bất ngờ là khi đọc tài liệu chính thức và các kiến trúc tham chiếu của AWS, hầu hết các giải pháp đều sử dụng **AWS Bedrock Knowledge Bases** thay vì tự triển khai và vận hành các cơ sở dữ liệu vector độc lập.

Điều này khiến mình đặt ra câu hỏi:

> *Nếu việc tự xây dựng pipeline chunking và quản lý vector database là cách tiếp cận phổ biến của RAG, tại sao AWS lại phát triển Bedrock Knowledge Bases?*

Sau khi nghiên cứu tài liệu chính thức và triển khai dịch vụ này trong một dự án **Medical AI Assistant**, mình nhận ra rằng AWS Bedrock Knowledge Bases không chỉ đơn thuần là một dịch vụ lưu trữ vector. Đây là một giải pháp được AWS quản lý hoàn toàn, tự động hóa toàn bộ quy trình RAG và biến một hạ tầng vốn phức tạp trở thành trải nghiệm **serverless**.

---

## Tự quản lý hạ tầng RAG – "Cơn ác mộng" trong vận hành

Ban đầu, mình nghĩ rằng xây dựng một hệ thống RAG chủ yếu chỉ là viết mã Python.

Ví dụ, để xử lý hàng nghìn tài liệu PDF và CSV về y tế, mình phải:

- Chia tài liệu thành các đoạn có ý nghĩa (semantic chunking).
- Tạo embedding bằng cách gọi các mô hình embedding.
- Triển khai và vận hành cơ sở dữ liệu vector như **Milvus** hoặc **Qdrant**.
- Xây dựng pipeline đồng bộ dữ liệu để cập nhật vector database.

Cách tiếp cận này hoạt động tốt đối với các dự án nhỏ. Tuy nhiên, khi lượng tri thức y tế ngày càng tăng, việc duy trì toàn bộ hạ tầng trở nên ngày càng phức tạp.

Đội ngũ kỹ thuật phải liên tục theo dõi:

- Tính sẵn sàng của vector database.
- Khả năng mở rộng dung lượng lưu trữ.
- Quá trình đồng bộ embedding.
- Các pipeline ingestion theo lịch.
- Việc bảo trì hạ tầng.

Thay vì tập trung phát triển ứng dụng, một lượng lớn thời gian lại dành cho việc vận hành hạ tầng.

Đó cũng chính là lý do các kiến trúc cloud hiện đại khuyến khích sử dụng **Managed Serverless Services**, cho phép nhà cung cấp dịch vụ đám mây quản lý hạ tầng, trong khi nhà phát triển chỉ cần tập trung vào bài toán nghiệp vụ.

---

## AWS Bedrock Knowledge Bases là gì?

Theo tài liệu của AWS, **Knowledge Bases for Amazon Bedrock** là một tính năng được AWS quản lý hoàn toàn, giúp kết nối các **Foundation Models** với nguồn dữ liệu của doanh nghiệp để triển khai mô hình **Retrieval-Augmented Generation (RAG)**.

Thay vì phải tự xây dựng từng bước trong quy trình, AWS tự động hóa toàn bộ pipeline ingestion.

Dịch vụ cung cấp:

- **Automated Chunking** – Tự động chia tài liệu thành các đoạn ngữ nghĩa phù hợp.
- **Automated Embedding** – Tự động tạo vector embedding bằng các mô hình như Amazon Titan hoặc Cohere.
- **Automated Indexing** – Tự động lưu trữ vector vào backend được quản lý như Amazon OpenSearch Serverless.

Việc sử dụng Bedrock Knowledge Bases đã thay đổi hoàn toàn góc nhìn của mình về phát triển hệ thống AI.

Trước đây mình cho rằng kỹ sư AI phải tự quản lý mọi công đoạn của pipeline dữ liệu.

Hiện nay mình nhận ra rằng việc vận hành hạ tầng nên được tự động hóa để kỹ sư có thể tập trung giải quyết các bài toán nghiệp vụ thay vì quản lý hệ thống.

---

## Quy trình làm việc với AWS Bedrock Knowledge Bases

Quá trình tích hợp trở nên đơn giản hơn rất nhiều.

### 1. Tải dữ liệu lên Amazon S3

Amazon S3 đóng vai trò là kho lưu trữ tri thức ban đầu.

Các hướng dẫn điều trị, phác đồ y khoa, tài liệu nghiên cứu và các tài liệu tham khảo khác chỉ cần được tải lên một S3 Bucket.

---

### 2. Khởi chạy Ingestion Job

Chỉ với một thao tác trên AWS Console hoặc một lời gọi API, dịch vụ sẽ tự động:

- Phát hiện các tệp mới được tải lên.
- Thực hiện semantic chunking.
- Sinh vector embedding.
- Cập nhật Knowledge Base.

Toàn bộ quá trình diễn ra tự động mà không cần dừng hệ thống hay can thiệp thủ công.

---

### 3. Truy xuất tri thức bằng LangChain

Ứng dụng có thể truy xuất ngữ cảnh liên quan thông qua `AmazonKnowledgeBasesRetriever` của LangChain.

Thay vì phải tự viết truy vấn hoặc quản lý hạ tầng tìm kiếm vector, nhà phát triển chỉ cần cung cấp:

- Knowledge Base ID.
- Cấu hình truy xuất (Retrieval Configuration).

Retriever hỗ trợ Hybrid Search và tích hợp trực tiếp với Amazon Bedrock.

---

## Vai trò của Amazon S3 trong kiến trúc

Ban đầu mình chỉ xem Amazon S3 như một dịch vụ lưu trữ đối tượng (Object Storage).

Tuy nhiên, sau khi tích hợp với Bedrock Knowledge Bases, mình nhận ra rằng S3 chính là **cổng tiếp nhận tri thức (Knowledge Gateway)** của toàn bộ hệ thống AI.

Mỗi khi bệnh viện cập nhật:

- Phác đồ điều trị mới.
- Danh mục thuốc mới.
- Hướng dẫn lâm sàng.
- Tài liệu chuyên môn.

Quản trị viên chỉ cần tải các tệp mới lên Amazon S3.

Không cần triển khai lại ứng dụng.

Không cần chỉnh sửa pipeline.

Sau khi chạy một Ingestion Job, hệ thống AI sẽ ngay lập tức có thể khai thác nguồn tri thức mới.

Sự kết hợp giữa Amazon S3 và AWS Bedrock Knowledge Bases tạo nên một pipeline quản lý tri thức hoàn toàn tự động và liên tục được cập nhật.

---

## Vì sao AWS Bedrock Knowledge Bases là "chân ái" của các kỹ sư AI?

Sau khi hoàn thành dự án, có một điều mình nhận thấy rất rõ:

Lợi ích lớn nhất của dịch vụ này là **loại bỏ gánh nặng quản lý hạ tầng**.

Nếu tự triển khai RAG, kỹ sư phải chịu trách nhiệm:

- Triển khai vector database.
- Giám sát hạ tầng.
- Xây dựng pipeline ingestion.
- Xử lý lỗi đồng bộ dữ liệu.
- Quản lý việc mở rộng hệ thống.

Với AWS Bedrock Knowledge Bases, tất cả những công việc này đều do AWS đảm nhiệm.

Thay vì lo lắng về hạ tầng, nhà phát triển có thể tập trung vào:

- Cải thiện chất lượng truy xuất thông tin.
- Tối ưu Prompt.
- Nâng cao quy trình nghiệp vụ.
- Mang đến trải nghiệm tốt hơn cho bệnh nhân.

Đối với các nhóm phát triển, điều này có thể tiết kiệm nhiều tuần triển khai và vận hành, đồng thời giúp kiến trúc hệ thống trở nên đơn giản hơn đáng kể.

---

## Kết luận

AWS Bedrock Knowledge Bases không chỉ đơn thuần là một cơ sở dữ liệu vector được quản lý.

Đây là một dịch vụ RAG được AWS quản lý hoàn toàn, tự động hóa toàn bộ quá trình ingest tài liệu, chunking, tạo embedding, lập chỉ mục và truy xuất dữ liệu.

Đối với các nhóm xây dựng ứng dụng AI doanh nghiệp, đặc biệt trong lĩnh vực y tế, dịch vụ này giúp loại bỏ phần lớn sự phức tạp trong vận hành của kiến trúc Retrieval-Augmented Generation, cho phép đội ngũ phát triển tập trung xây dựng các ứng dụng AI thông minh thay vì dành thời gian quản lý hạ tầng.

---

## Tài liệu tham khảo

- AWS. *Knowledge Bases for Amazon Bedrock.*  
  https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html

- LangChain. *Amazon Knowledge Bases Integrations.*  
  https://python.langchain.com/docs/integrations/retrievers/bedrock/