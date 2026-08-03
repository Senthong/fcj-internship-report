---
title: "Worklog Tuần 8"
date: 2026-07-20
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Mục tiêu tuần 8:

* Thiết lập pipeline CI bằng GitHub Actions: lint Python, biên dịch model dbt, xác thực Terraform — chạy trên mỗi lần push/PR.
* Chạy kiểm thử hồi quy end-to-end toàn bộ pipeline (S3 → Redshift → dbt → mart) để xác nhận không có gì bị hỏng sau khi ghép nối tất cả các phần lại.
* Đánh giá hệ thống một cách khách quan so với 6 mục tiêu ban đầu, ghi nhận ưu điểm và hạn chế cho báo cáo.
* Hoàn thiện báo cáo tốt nghiệp: nội dung viết, sơ đồ, đoạn mã minh hoạ, và phụ lục toàn văn DAG.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Viết `.github/workflows/ci.yml`: job `lint-python` (`black --check`, `flake8`) <br> - Viết job `dbt-compile` (`dbt deps && dbt compile --target dev` với host giả, không cần DB thật)     | 20/07/2026   | 20/07/2026      | GitHub Actions docs |
| 3   | - Viết job `terraform-validate` (`terraform init -backend=false`, `terraform validate`, `terraform fmt -check`) <br> - **Kiểm thử:** mở PR với lỗi lint cố ý và một khối Terraform sai, xác nhận CI báo lỗi cả hai | 21/07/2026   | 21/07/2026      | |
| 4   | - Chạy kiểm thử hồi quy end-to-end: `terraform apply` từ đầu → Docker Compose up → trigger `ecom_full_pipeline_dag` → kiểm tra cả 4 bảng mart có dữ liệu đúng và cả 46 test dbt đều pass       | 22/07/2026   | 22/07/2026      | |
| 5   | - Viết Chương 5 (Kết quả và Đánh giá): tổng hợp số liệu cuối cùng (11 model dbt, 46 test, DAG 7 task, 4 nhóm tài nguyên Terraform, 3 job CI) đối chiếu 6 mục tiêu ban đầu <br> - Ghi nhận trung thực các hạn chế (chưa có incremental model, chỉ dùng LocalExecutor, chưa cảnh báo chủ động, chưa có CD, chưa có unit test, chưa có tầng BI) | 23/07/2026   | 23/07/2026      | |
| 6   | - Viết Chương 6 (Kết luận và Hướng phát triển) <br> - Ghép hoàn chỉnh báo cáo: chương cơ sở lý thuyết, sơ đồ kiến trúc, đoạn mã minh hoạ, tài liệu tham khảo, phụ lục DAG <br> - Đẩy mã nguồn cuối cùng lên GitHub repo công khai và đọc soát lại toàn bộ báo cáo | 24/07/2026   | 25/07/2026 | |


### Kết quả đạt được tuần 8:

* Hoàn thành `.github/workflows/ci.yml` với 3 job độc lập chạy song song trên mỗi lần push/PR vào `main`/`develop`: `lint-python`, `dbt-compile`, `terraform-validate`.

* Xác nhận `dbt compile` đủ để phát hiện phần lớn lỗi cú pháp SQL/Jinja và tham chiếu `ref()`/`source()` bị hỏng mà không cần kết nối Redshift thật — giữ pipeline CI nhanh và không tốn chi phí hạ tầng.

* Kiểm chứng CI thực sự phát hiện được lỗi: cố tình commit một tệp Python định dạng sai và một khối Terraform có lỗi cú pháp, xác nhận cả `lint-python` và `terraform-validate` đều báo fail PR check như mong đợi.

* Chạy kiểm thử hồi quy end-to-end hoàn chỉnh từ đầu (`terraform destroy` → `apply` → Docker Compose → trigger DAG) và xác nhận toàn bộ pipeline vẫn hoạt động đúng sau 7 tuần thay đổi tích luỹ — không có hồi quy nào ở idempotency, quality gate, hay cơ chế rollback transaction.

* Tổng hợp quy mô cuối cùng của dự án cho báo cáo: 11 mô hình dbt (7 staging + 4 mart), 46 bài kiểm thử chất lượng dữ liệu, DAG Airflow 7 task, 4 nhóm tài nguyên AWS được Terraform quản lý, và pipeline CI 3 job.

* Viết phần hạn chế một cách trung thực: bảng mart vẫn full-refresh (chưa incremental), Airflow chỉ chạy `LocalExecutor`, cảnh báo lỗi mới dừng ở mức ghi log (chưa có Slack/email), chưa có bước Continuous Deployment tự động, chưa có unit test `pytest` cho script ingestion, và chưa có tầng BI/dashboard — tất cả đều được nêu rõ là hướng phát triển tương lai thay vì che giấu.

* Hoàn thiện và soát lỗi toàn bộ báo cáo tốt nghiệp (6 chương + phụ lục toàn văn DAG), đẩy phiên bản mã nguồn cuối cùng lên repository công khai được nhắc đến trong phần lời cam đoan của báo cáo.
