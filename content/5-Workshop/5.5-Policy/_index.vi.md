---
title : "Cấu hình Quản trị & Phân quyền"
date : 2026-07-27
weight : 5
chapter : false
pre : " <b> 5.5 </b> "
---

Mục này trình bày các cơ chế quản trị (governance) được thiết lập xuyên suốt dự án, chia thành hai lớp: lớp thứ nhất là quản trị ở tầng biến đổi dữ liệu - cách dbt được cấu hình tập trung để toàn bộ model tuân theo cùng một quy ước về schema, vật chất hóa và đặt tên; lớp thứ hai là quản trị ở tầng hạ tầng và quy trình phát triển - kiểm soát ai/dịch vụ nào được truy cập tài nguyên AWS nào, và mã nguồn được kiểm tra tự động ra sao trước khi gộp vào nhánh chính. Phần cuối mục này cũng khép lại một điểm còn để ngỏ từ mục 5.3.1: chính sách Full Access tạm thời gán cho Gateway VPC Endpoint trong giai đoạn khởi tạo.

## 1. Cấu hình quản trị tập trung: `dbt_project.yml`

Thay vì khai báo `materialized`, `schema`, `tags` lặp lại ở từng model bằng khối `{{ config(...) }}`, phần lớn các thuộc tính này được thiết lập một lần tại `dbt_project.yml` theo cấu trúc phân cấp thư mục, áp dụng mặc định cho toàn bộ model nằm trong thư mục tương ứng:

```yaml
name: 'ecom_pipeline'
version: '1.0.0'
config-version: 2

profile: 'ecom_pipeline'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

# 
# Model configs per layer
# 
models:
  ecom_pipeline:
    staging:
      +materialized: view
      +schema: staging
      +tags: ["staging"]

    intermediate:
      +materialized: ephemeral
      +tags: ["intermediate"]

    mart:
      +materialized: table
      +schema: mart
      +tags: ["mart"]

      finance:
        +materialized: table
        revenue_daily:
          +sort: ["order_date"]
          +dist: "order_date"

      customers:
        +materialized: table
        fct_customer_cohorts:
          +sort: ["cohort_month"]

      operations:
        +materialized: table

vars:
  # Used to filter data in development so dbt runs fast locally
  start_date: '2016-01-01'
  # Override in prod: dbt run --vars '{"start_date": "2016-01-01"}'
```

Khối `models.ecom_pipeline` thể hiện rõ ba tầng dữ liệu đã đề cập xuyên suốt mục 5.4: `staging` mặc định vật chất hóa dưới dạng `view`, gắn schema `staging` và tag `staging`; `mart` mặc định vật chất hóa dưới dạng `table`, gắn schema `mart` và tag `mart`, với ba khối con `finance`/`customers`/`operations` tương ứng ba thư mục nghiệp vụ đã trình bày ở mục 5.4.2 - riêng `revenue_daily` và `fct_customer_cohorts` còn được khai báo thêm `sort`/`dist` key ngay tại đây; một số model như `seller_performance` chọn khai báo lại `dist` trực tiếp trong khối `{{ config(...) }}` của chính file model thay vì tại đây, dbt cho phép cấu hình ở cấp file ghi đè cấu hình mặc định ở cấp thư mục khi cần tùy biến riêng cho một model cụ thể.

Đáng chú ý là dự án đã khai báo sẵn tầng `intermediate` với `+materialized: ephemeral` (vật chất hóa dưới dạng CTE lồng vào truy vấn gọi nó, không tạo view hay table riêng trong Redshift) - tầng trung gian dành cho các phép biến đổi phức tạp cần tái sử dụng giữa nhiều Data Mart mà không muốn tạo thêm đối tượng vật lý trong kho dữ liệu. Tại thời điểm hiện tại, thư mục `models/intermediate/` chưa có model nào; đây là cấu hình dự phòng (reserved) cho giai đoạn mở rộng tiếp theo, khi logic dùng chung giữa các Data Mart trở nên phức tạp hơn mức các Staging Model hiện tại có thể đảm nhiệm.

Khối `vars.start_date` là biến toàn cục được bốn Data Mart (`revenue_daily`, `category_revenue_monthly`, `fct_customer_cohorts`, `seller_performance`) cùng tham chiếu qua `{{ var("start_date") }}` để giới hạn tập dữ liệu xử lý. Giá trị mặc định `2016-01-01` trùng với thời điểm sớm nhất của bộ dữ liệu Olist nên trong môi trường `prod` không cắt bớt dữ liệu nào; giá trị thực sự phát huy tác dụng khi nhà phát triển cần rút ngắn thời gian chạy `dbt run` cục bộ bằng cách ghi đè qua tham số dòng lệnh, ví dụ `dbt run --vars '{"start_date": "2024-01-01"}'`, mà không phải sửa mã nguồn của bất kỳ model nào.

## 2. Quy ước đặt tên schema theo môi trường

Một trong những rủi ro phổ biến khi nhiều người cùng phát triển trên một dự án dbt là nhà phát triển chạy `dbt run` từ máy cá nhân trong lúc thử nghiệm và vô tình ghi đè lên đúng schema `staging`/`mart` mà tác vụ Airflow chạy hằng ngày đang phục vụ báo cáo. Dự án phòng tránh rủi ro này bằng một macro tùy chỉnh, `macros/generate_schema_name.sql`, ghi đè hành vi đặt tên schema mặc định của dbt:

```sql
-- macros/generate_schema_name.sql
-- 
-- Custom schema naming convention:
--   dev  target: analytics_dev__staging, analytics_dev__mart
--   prod target: staging, mart  (no prefix)

{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}

    {%- if target.name == 'prod' -%}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ custom_schema_name | trim }}
        {%- endif -%}

    {%- else -%}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ default_schema }}_{{ custom_schema_name | trim }}
        {%- endif -%}
    {%- endif -%}

{%- endmacro %}
```

Kết hợp với hai target đã khai báo trong `profiles.yml` (`dev` dùng schema mặc định `analytics_dev`, `prod` dùng schema mặc định `analytics`), macro trên tạo ra hai kết quả khác nhau tùy vào target đang chạy:

| Target đang chạy (`--target`) | `custom_schema_name` khai báo trong model | Schema thực tế được ghi vào |
| :--- | :--- | :--- |
| `prod` | `staging` | `staging` |
| `prod` | `mart` | `mart` |
| `dev` | `staging` | `analytics_dev_staging` |
| `dev` | `mart` | `analytics_dev_mart` |

Lưu ý: dòng chú thích ở đầu file macro ghi ví dụ `analytics_dev__staging` (hai dấu gạch dưới), nhưng theo đúng biểu thức nối chuỗi trong thân macro (`{{ default_schema }}_{{ custom_schema_name }}`), kết quả thực tế chỉ có một dấu gạch dưới giữa `analytics_dev` và `staging`, đúng như thể hiện trong bảng trên - đây là một chú thích mô tả chưa khớp tuyệt đối với hành vi thật của macro, không ảnh hưởng đến chức năng nhưng nên được sửa lại trong lần cập nhật mã nguồn tiếp theo để tránh gây nhầm lẫn cho người đọc sau này.

Khi chạy với `--target prod` - đúng như biến `DBT_TARGET: "prod"` mà Airflow truyền vào ở mục 5.4.3 - macro bỏ qua tiền tố mặc định và trả về đúng tên schema đã khai báo trong `dbt_project.yml` (`staging`, `mart`), khớp với hai schema đã tạo thủ công bằng `CREATE SCHEMA` ở mục 5.3.1. Ngược lại, khi một nhà phát triển chạy `dbt run` từ máy cá nhân mà không truyền `--target prod` (mặc định rơi về `dev` theo khai báo `target: "{{ env_var('DBT_TARGET', 'dev') }}"` trong `profiles.yml`), toàn bộ model sẽ được ghi vào các schema có tiền tố `analytics_dev_`, tách biệt hoàn toàn khỏi dữ liệu production. Nhờ cơ chế này, việc phát triển và thử nghiệm model mới không bao giờ vô tình ảnh hưởng đến dữ liệu mà pipeline tự động đang phục vụ, mà không đòi hỏi nhà phát triển phải nhớ tự tay đổi tên schema mỗi lần chạy thử.


## 3. Quy ước đặt tên model và quản lý phụ thuộc package

Bên cạnh cấu hình schema, dự án duy trì một số quy ước đặt tên nhất quán giúp mã nguồn dễ định vị khi số lượng model tăng lên:

| Quy ước | Áp dụng cho | Ví dụ |
| :--- | :--- | :--- |
| Tiền tố `stg_` | Toàn bộ Staging Model | `stg_orders`, `stg_sellers` |
| Tiền tố `fct_` | Bảng sự kiện/fact ở tầng Mart | `fct_customer_cohorts` |
| Tên mô tả trực tiếp số liệu | Các bảng tổng hợp (aggregate) ở tầng Mart | `revenue_daily`, `seller_performance` |
| Một thư mục con cho mỗi nhóm nghiệp vụ | Tầng Mart | `mart/finance/`, `mart/customers/`, `mart/operations/` |
| Gắn tag theo tầng và nhóm nghiệp vụ | Mọi model | `tags: ['mart', 'finance', 'daily']` |

Quy ước gắn tag không chỉ mang tính mô tả mà còn được khai thác trực tiếp trong vận hành: hai tác vụ `dbt_run_staging`/`dbt_run_mart` ở mục 5.4.3 chạy bằng `--select staging` và `--select mart` chính là dựa vào cơ chế lựa chọn theo đường dẫn thư mục (trùng với tag cùng tên) của dbt, cho phép Airflow điều khiển chính xác tầng nào chạy trước, tầng nào chạy sau mà không cần liệt kê tên từng model.

Phụ thuộc bên ngoài của dự án được quản lý qua `packages.yml`, khai báo khoảng phiên bản chấp nhận được thay vì cố định một phiên bản duy nhất:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
  - package: dbt-labs/audit_helper
    version: [">=0.10.0", "<1.0.0"]
```

Khi chạy `dbt deps`, phiên bản cụ thể được dbt phân giải trong khoảng cho phép sẽ được ghi lại vào `package-lock.yml` (`dbt_utils` phân giải về `1.4.1`, `audit_helper` về `0.14.0`), tương tự vai trò của tệp lockfile trong các hệ quản lý gói khác - đảm bảo mọi môi trường (máy nhà phát triển, container CI, container Airflow production) đều cài đặt đúng cùng một phiên bản package đã được kiểm chứng, tránh tình trạng một bản cập nhật package ngoài ý muốn làm thay đổi hành vi của các test `dbt_utils.expression_is_true` đang được dùng rộng rãi ở mục 5.4.3.

## 4. Kiểm soát truy cập hạ tầng: từ IAM Role đến khoảng trống trong chính sách VPC Endpoint

Như đã trình bày ở mục 5.2, quyền truy cập S3 dành cho Redshift được quản lý qua một IAM Role riêng, tách biệt khỏi quyền của người vận hành, với chính sách giới hạn đúng hai hành động và đúng hai bucket của dự án:

```hcl
resource "aws_iam_policy" "redshift_s3_policy" {
  name = "${local.project}-redshift-s3-policy-${local.environment}"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:ListBucket"]
        Resource = [
          aws_s3_bucket.raw.arn,
          "${aws_s3_bucket.raw.arn}/*",
          aws_s3_bucket.processed.arn,
          "${aws_s3_bucket.processed.arn}/*",
        ]
      }
    ]
  })
}
```

Đây là lớp kiểm soát quyền ở phía IAM - quyết định *dịch vụ nào được phép gọi API S3 nào*. Tuy nhiên, mục 5.3.1 đã đề cập đến một lớp kiểm soát khác, độc lập với IAM: chính sách gắn trực tiếp vào Gateway VPC Endpoint, quyết định *những request nào được phép đi qua đường kết nối riêng đó*, và đã nêu rõ chính sách Full Access được chọn tạm thời trong giai đoạn khởi tạo sẽ được thu hẹp lại tại mục 5.5 này.

Khi rà soát lại toàn bộ mã nguồn Terraform (`infrastructure/terraform/main.tf`) ở thời điểm hiện tại, dự án **chưa có** tài nguyên `aws_vpc_endpoint` nào được định nghĩa - toàn bộ Gateway VPC Endpoint mô tả ở mục 5.3.1 mới chỉ tồn tại dưới dạng cấu hình thao tác thủ công qua AWS Console, chưa được đưa vào Infrastructure as Code. Đây là một khoảng trống cần bổ sung để mã Terraform phản ánh đúng và đầy đủ hạ tầng đang chạy, đồng thời hoàn tất lời hứa đã nêu ở mục 5.3.1. Đoạn cấu hình khuyến nghị bổ sung như sau:

```hcl
resource "aws_vpc_endpoint" "s3_gateway" {
  vpc_id            = var.vpc_id
  service_name      = "com.amazonaws.${var.aws_region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = var.private_route_table_ids

  # Thu hẹp từ Full Access xuống đúng hai bucket phục vụ dự án,
  # thay vì cho phép truy cập mọi bucket S3 trong tài khoản AWS.
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [
          aws_s3_bucket.raw.arn,
          "${aws_s3_bucket.raw.arn}/*",
          aws_s3_bucket.processed.arn,
          "${aws_s3_bucket.processed.arn}/*",
        ]
      }
    ]
  })

  tags = local.tags
}
```

So với chính sách Full Access ban đầu, điểm khác biệt cốt lõi nằm ở khối `Resource`: thay vì để trống (ngầm định cho phép mọi bucket S3 trong tài khoản AWS), chính sách trên chỉ liệt kê ARN của đúng hai bucket `raw` và `processed` của dự án. Về mặt hiệu quả bảo mật, việc thu hẹp này đóng vai trò như một lớp phòng vệ theo chiều sâu (defense in depth) bổ sung cho IAM Role đã có: cho dù trong tương lai một tài nguyên khác trong cùng VPC (ví dụ một EC2 instance chạy dịch vụ không liên quan) được gán nhầm quyền `AmazonS3FullAccess`, Gateway VPC Endpoint vẫn chỉ cho phép lưu lượng đi đến hai bucket đã khai báo, không mở đường truy cập tới các bucket khác trong cùng tài khoản.

## 5. Governance qua CI/CD (GitHub Actions)

Song song với các cơ chế trên, dự án dùng GitHub Actions (`.github/workflows/ci.yml`) làm lớp kiểm soát chất lượng mã nguồn trước khi thay đổi được gộp vào nhánh `main`, kích hoạt trên mỗi lần `push` vào `main`/`develop` và mỗi Pull Request nhắm tới `main`. Ba công việc (job) sau luôn chạy:

| Job | Nội dung kiểm tra |
| :--- | :--- |
| `lint-python` | Định dạng mã nguồn Python bằng `black --check`, thứ tự import bằng `isort --check-only`, và lỗi cú pháp/phong cách bằng `flake8`, áp dụng cho `ingestion/` và `airflow/` |
| `dbt-compile` | Cài `dbt-core==1.7.3`, `dbt-redshift==1.7.1`, chạy `dbt deps` rồi `dbt compile --target dev` với thông tin kết nối Redshift giả (`dummy.host`) - việc biên dịch chỉ nhằm phát hiện lỗi cú pháp SQL/Jinja trong các model, không cần kết nối thật tới Redshift |
| `terraform-validate` | `terraform init -backend=false`, `terraform validate` và `terraform fmt -check` trên `infrastructure/terraform`, đảm bảo mã hạ tầng luôn ở trạng thái hợp lệ và được định dạng nhất quán |

Việc job `dbt-compile` sử dụng thông tin kết nối giả (`REDSHIFT_HOST: "dummy.host"`, `REDSHIFT_USER: "ci_user"`) là một lựa chọn kỹ thuật hợp lý: mục tiêu của bước này chỉ là xác nhận cú pháp SQL và Jinja hợp lệ cũng như `ref()`/`source()` trỏ đúng đối tượng đã khai báo, nên không cần thiết phải mở kết nối mạng tới Redshift Serverless production, giúp pipeline CI chạy nhanh và không phụ thuộc vào thông tin xác thực bí mật khi chỉ đang xử lý một Pull Request.

Tệp cấu hình còn giữ sẵn một job thứ tư ở dạng chú thích (comment), dự kiến để tự động chạy `dbt run --target prod` mỗi khi có thay đổi được gộp vào `main`:

```yaml
  #  Deploy to prod (on main push only) 
  # Uncomment and configure when ready for automated deploys
  # deploy:
  #   name: Deploy dbt to Production
  #   needs: [lint-python, dbt-compile]
  #   if: github.ref == 'refs/heads/main'
  #   runs-on: ubuntu-latest
  #   environment: production
```

Việc job triển khai tự động này chưa được kích hoạt là một quyết định thận trọng hợp lý ở giai đoạn hiện tại của dự án: tự động chạy `dbt run --target prod` ngay khi mã nguồn được gộp vào `main`, trong khi chưa có bước phê duyệt thủ công hoặc môi trường staging trung gian, có thể khiến một thay đổi model chưa được kiểm chứng đầy đủ ảnh hưởng trực tiếp đến dữ liệu production. Đây là hạng mục phù hợp để kích hoạt trong giai đoạn kế tiếp, sau khi bổ sung cơ chế phê duyệt (ví dụ GitHub Environment với required reviewers) cho môi trường `production` đã được khai báo sẵn trong đoạn cấu hình trên.


## 6. Ghi chú về tính nhất quán giữa tài liệu và mã nguồn dự án

Việc duy trì tài liệu mô tả đúng với những gì mã nguồn thực sự triển khai cũng là một hình thức quản trị, thường bị xem nhẹ so với các cơ chế kỹ thuật. Trong quá trình rà soát phục vụ hai mục 5.4 và 5.5, có hai điểm chênh lệch giữa `README.md` và trạng thái mã nguồn hiện tại đáng được ghi nhận để cập nhật ở lần soát xét tài liệu tiếp theo: mục **Project Structure** của `README.md` liệt kê thư mục `scripts/` như một phần cấu trúc dự án, nhưng thư mục này chưa tồn tại trong mã nguồn hiện tại; mục **Skills Demonstrated** liệt kê "Implementing incremental dbt models for large tables", trong khi toàn bộ bốn Data Mart ở mục 5.4.2 hiện đều được cấu hình `materialized='table'` (full refresh mỗi lần chạy), chưa có model nào sử dụng chiến lược `materialized='incremental'`. Đây đều là các hạng mục thuộc định hướng phát triển tiếp theo hơn là tính năng đã hoàn thiện, nên được điều chỉnh trong tài liệu để phản ánh chính xác phạm vi đã triển khai tại thời điểm báo cáo này được lập.