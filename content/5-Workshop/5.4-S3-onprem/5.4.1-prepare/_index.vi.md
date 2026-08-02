---
title : "5.4.1 - Chuẩn bị Tầng Staging (Ingestion & Staging Models)"
date : 2026-07-23
weight : 1
chapter : false
pre : " <b> 5.4.1 </b> "
---

Mục này trình bày hai nhóm công việc diễn ra kế tiếp nhau: thứ nhất, cơ chế đưa dữ liệu thô từ Amazon S3 vào schema `staging` trên Amazon Redshift; thứ hai, cách dbt Core chuẩn hóa các bảng thô đó thành tập hợp các Staging Model - lớp trung gian duy nhất mà toàn bộ Data Mart ở mục 5.4.2 sẽ tham chiếu tới thông qua hàm `ref()`. Việc tách hai bước này (nạp dữ liệu thô nguyên trạng, rồi mới làm sạch bằng SQL) chính là bản chất của kiến trúc ELT đã nêu ở mục 5.1: Redshift, chứ không phải một tầng trung gian ngoài kho dữ liệu, đảm nhận toàn bộ việc biến đổi.

## 1. Nạp dữ liệu thô vào Redshift bằng lệnh COPY

Việc nạp dữ liệu không được thực hiện thủ công mà thông qua script `ingestion/load_to_redshift.py`, được Airflow gọi lại ở task `load_staging_to_redshift` (trình bày chi tiết ở mục 5.4.3). Script này không nạp trực tiếp theo kiểu "đọc file rồi insert từng dòng", mà sử dụng lệnh `COPY` của Redshift - phương thức nạp hàng loạt hiệu quả nhất trên nền tảng này vì tận dụng được kiến trúc xử lý song song (MPP) để đọc trực tiếp file trên S3 vào nhiều node cùng lúc.

Toàn bộ 8 bảng thô cần nạp được khai báo tập trung dưới dạng một danh sách các đối tượng `TableSpec` (Python dataclass), mỗi đối tượng gắn liền tên file CSV nguồn, tên bảng đích, câu lệnh `CREATE TABLE` kèm kiểu dữ liệu, và tùy chọn khóa phân phối (`DISTKEY`) / khóa sắp xếp (`SORTKEY`) tối ưu cho Redshift. Cách tổ chức này giúp việc thêm một bảng nguồn mới chỉ đơn giản là khai báo thêm một `TableSpec`, thay vì viết lại logic nạp dữ liệu:

```python
# ingestion/load_to_redshift.py
@dataclass
class TableSpec:
    """Defines how each CSV maps to a Redshift table."""
    csv_file: str
    table_name: str
    columns: str
    ddl: str
    sort_key: Optional[str] = None
    dist_key: Optional[str] = None

TABLE_SPECS = [
    TableSpec(
        csv_file="olist_orders_dataset.csv",
        table_name="raw_orders",
        columns="order_id, customer_id, order_status, order_purchase_timestamp, "
                "order_approved_at, order_delivered_carrier_date, "
                "order_delivered_customer_date, order_estimated_delivery_date",
        sort_key="order_purchase_timestamp",
        dist_key="order_id",
        ddl="""
        CREATE TABLE IF NOT EXISTS {schema}.raw_orders (
            order_id                       VARCHAR(64)  NOT NULL,
            customer_id                    VARCHAR(64),
            order_status                   VARCHAR(50),
            order_purchase_timestamp       TIMESTAMP,
            order_approved_at              TIMESTAMP,
            order_delivered_carrier_date   TIMESTAMP,
            order_delivered_customer_date  TIMESTAMP,
            order_estimated_delivery_date  TIMESTAMP,
            _loaded_at                     TIMESTAMP DEFAULT GETDATE()
        )
        DISTSTYLE KEY DISTKEY(order_id)
        SORTKEY(order_purchase_timestamp);
        """,
    ),
    # ... 7 TableSpec còn lại: raw_customers, raw_order_items,
    #     raw_order_payments, raw_order_reviews, raw_products,
    #     raw_sellers, raw_category_translation
]
```

Bảng dưới đây tóm tắt đầy đủ 8 bảng thô được script tạo và nạp vào schema `staging`:

| Bảng đích | Tệp CSV nguồn | Khóa phân phối (DISTKEY) | Ghi chú |
| :--- | :--- | :--- | :--- |
| `raw_orders` | `olist_orders_dataset.csv` | `order_id` | Sắp xếp theo `order_purchase_timestamp` để tối ưu truy vấn theo thời gian |
| `raw_order_items` | `olist_order_items_dataset.csv` | `order_id` | Một dòng cho mỗi mặt hàng trong đơn |
| `raw_customers` | `olist_customers_dataset.csv` | `customer_id` | Sắp xếp và phân phối cùng khóa `customer_id` |
| `raw_sellers` | `olist_sellers_dataset.csv` | `DISTSTYLE ALL` | Bảng nhỏ, nhân bản trên mọi node để tăng tốc JOIN |
| `raw_products` | `olist_products_dataset.csv` | `DISTSTYLE ALL` | Bảng nhỏ, nhân bản trên mọi node |
| `raw_order_payments` | `olist_order_payments_dataset.csv` | `order_id` | Một đơn có thể có nhiều dòng thanh toán (trả góp) |
| `raw_order_reviews` | `olist_order_reviews_dataset.csv` | `order_id` | |
| `raw_category_translation` | `product_category_name_translation.csv` | `DISTSTYLE ALL` | Bảng tra cứu (lookup), dùng để dịch tên danh mục sản phẩm |

Mỗi bảng đều có thêm cột `_loaded_at TIMESTAMP DEFAULT GETDATE()` - đây chính là cột được khai báo làm `loaded_at_field` trong `sources.yml` để dbt kiểm tra độ mới (freshness) của dữ liệu, với ngưỡng cảnh báo sau 25 giờ và ngưỡng lỗi sau 49 giờ kể từ lần nạp gần nhất. Ngưỡng này được chọn có chủ đích rộng hơn chu kỳ chạy hằng ngày một khoảng an toàn, để một lần chạy pipeline bị trễ không ngay lập tức bị coi là sự cố.

Điểm đáng chú ý là danh sách 8 bảng trên **không bao gồm** dữ liệu định vị địa lý (`olist_geolocation_dataset.csv`), mặc dù tệp này vẫn được script `ingestion/ingest_olist_to_s3.py` tải về và đẩy lên S3 cùng các tệp khác. Đây là một quyết định thiết kế có chủ đích, không phải thiếu sót: dữ liệu geolocation có khối lượng lớn nhưng ít được các Data Mart ở mục 5.4.2 sử dụng trực tiếp, nên được giữ lại ở tầng Bronze trên S3 và chỉ phục vụ truy vấn khám phá qua AWS Glue Crawler / Amazon Athena như đã mô tả ở mục 5.1, thay vì tốn chi phí lưu trữ và nạp vào Redshift.

Về cơ chế nạp, mỗi bảng được xử lý theo mẫu **TRUNCATE rồi COPY toàn bộ** (full refresh), không nạp tăng trưởng (incremental):

```python
def truncate_and_load(cursor, spec: TableSpec, s3_prefix: str, schema: str = STAGING_SCHEMA):
    table = f"{schema}.{spec.table_name}"
    s3_path = f"s3://{S3_BUCKET}/{s3_prefix}{spec.csv_file}"

    cursor.execute(spec.ddl.format(schema=schema))
    cursor.execute(f"TRUNCATE TABLE {table};")

    copy_sql = f"""
    COPY {table} ({spec.columns})
    FROM '{s3_path}'
    {_auth_clause()}
    REGION '{AWS_REGION}'
    FORMAT AS CSV
    QUOTE AS '"'
    IGNOREHEADER 1
    TIMEFORMAT 'auto'
    DATEFORMAT 'auto'
    BLANKSASNULL
    EMPTYASNULL
    MAXERROR 10;
    """
    cursor.execute(copy_sql)
```

Việc chọn `TRUNCATE + COPY` toàn bộ thay vì nạp tăng trưởng phù hợp với đặc điểm của bộ dữ liệu Olist - vốn là một tập dữ liệu tĩnh, có kích thước vừa phải (hơn 100.000 đơn hàng), nên chi phí nạp lại toàn bộ mỗi ngày là chấp nhận được và giúp logic nạp dữ liệu đơn giản, dễ kiểm chứng. Toàn bộ quá trình nạp 8 bảng được bọc trong một transaction duy nhất (`conn.autocommit = False`); nếu bất kỳ bảng nào nạp lỗi, `conn.rollback()` được gọi để hoàn tác toàn bộ, tránh tình trạng schema `staging` chỉ được cập nhật một phần.

Về xác thực với S3, hàm `_auth_clause()` ưu tiên sử dụng `IAM_ROLE` - chính là IAM Role đã được Terraform khởi tạo và gắn vào Redshift Namespace ở mục 5.3.1 - và chỉ dùng cặp Access Key/Secret Key làm phương án dự phòng. Cách làm này nhất quán với nguyên tắc đặc quyền tối thiểu đã đề cập ở mục 5.2: câu lệnh `COPY` chạy bên trong Redshift không cần mang theo khóa truy cập tĩnh, mà mượn quyền hạn tạm thời thông qua vai trò dịch vụ.

![chạy `load_to_redshift.py`](image-1.png)
## 2. Định nghĩa các dbt Staging Model

Sau khi dữ liệu thô đã nằm trong schema `staging`, dbt đảm nhận phần còn lại của tầng Silver: chuẩn hóa kiểu dữ liệu, đặt lại tên cột theo quy ước rõ nghĩa, lọc bản ghi không hợp lệ và tính toán một số cột phái sinh phục vụ trực tiếp cho tầng Data Mart. Dự án định nghĩa **7 Staging Model**, mỗi model tương ứng một bảng thô, đặt trong thư mục `models/staging/` và đặt tên theo quy ước `stg_<tên_thực_thể>`:

| Model | Nguồn (`source`) | Logic biến đổi chính |
| :--- | :--- | :--- |
| `stg_customers` | `raw_customers` | Chuẩn hóa mã bưu chính về 8 ký tự (`lpad`), viết hoa chữ cái đầu tên thành phố, viết hoa toàn bộ mã bang |
| `stg_sellers` | `raw_sellers` | Áp dụng cùng quy tắc chuẩn hóa địa chỉ như `stg_customers` |
| `stg_order_items` | `raw_order_items` | Ép kiểu `decimal(10,2)` cho giá và phí vận chuyển, tính cột `total_item_revenue`, loại bỏ dòng có giá trị âm |
| `stg_order_payments` | `raw_order_payments` | Chuẩn hóa `payment_type` về chữ thường, loại bỏ bản ghi có `payment_value` âm |
| `stg_order_reviews` | `raw_order_reviews` | Phân loại điểm đánh giá thành `positive` / `neutral` / `negative`, tính số ngày phản hồi đánh giá (`days_to_answer`) |
| `stg_products` | `raw_products`, `raw_category_translation` | JOIN với bảng dịch danh mục sản phẩm sang tiếng Anh, tính khối lượng thể tích quy đổi (`volumetric_weight_g`) |
| `stg_orders` | `raw_orders` | Chuẩn hóa trạng thái đơn hàng, tính các cờ `is_delivered`, `is_late_delivery` và số ngày giao trễ |

`stg_orders` là model phức tạp nhất, vì phần lớn logic nghiệp vụ về vòng đời đơn hàng - vốn được nhiều Data Mart ở mục 5.4.2 tái sử dụng - được tính toán một lần duy nhất tại đây thay vì lặp lại ở từng model tầng Gold:

```sql
-- models/staging/stg_orders.sql
-- 
-- Cleans and standardises the raw_orders table.
-- Applies type casting, null handling, and adds derived columns.
-- Materialized as view (cheap, always fresh).

with source as (
    select * from {{ source('olist_raw', 'raw_orders') }}
),

cleaned as (
    select
        -- Keys
        order_id,
        customer_id,

        -- Enums
        lower(trim(order_status))                         as order_status,

        -- Timestamps — cast and normalise
        order_purchase_timestamp::timestamp               as ordered_at,
        order_approved_at::timestamp                      as approved_at,
        order_delivered_carrier_date::timestamp           as shipped_at,
        order_delivered_customer_date::timestamp          as delivered_at,
        order_estimated_delivery_date::timestamp          as estimated_delivery_at,

        -- Derived
        date_trunc('day', order_purchase_timestamp)::date as order_date,
        date_trunc('week', order_purchase_timestamp)::date as order_week,
        date_trunc('month', order_purchase_timestamp)::date as order_month,

        -- Is the order actually delivered?
        case
            when lower(order_status) = 'delivered'
             and order_delivered_customer_date is not null
            then true
            else false
        end                                               as is_delivered,

        -- Was delivery late?
        case
            when order_delivered_customer_date is not null
             and order_estimated_delivery_date is not null
             and order_delivered_customer_date > order_estimated_delivery_date
            then true
            else false
        end                                               as is_late_delivery,

        -- Days late (negative = early)
        datediff(
            'day',
            order_estimated_delivery_date,
            order_delivered_customer_date
        )                                                 as days_late,

        _loaded_at

    from source
    where order_id is not null
)

select * from cleaned
```

Ba cột phái sinh `order_date`/`order_week`/`order_month` được tính sẵn ở đây để mọi Data Mart cần gộp nhóm theo thời gian (`revenue_daily`, `category_revenue_monthly`, `fct_customer_cohorts`) đều dùng chung một định nghĩa "ngày đặt hàng", tránh tình trạng mỗi model tự viết `date_trunc` theo cách khác nhau. Tương tự, hai cờ `is_delivered` và `is_late_delivery` được định nghĩa tường minh và duy nhất tại đây, thay vì để từng Data Mart tự suy luận lại điều kiện so sánh ngày giao hàng thực tế với ngày giao hàng dự kiến.

`stg_products` minh họa một dạng logic khác - kết hợp hai nguồn dữ liệu ngay tại tầng Staging để tạo ra một bảng "đủ dùng" cho tầng dưới:

```sql
-- models/staging/stg_products.sql
with products as (
    select * from {{ source('olist_raw', 'raw_products') }}
),
translations as (
    select * from {{ source('olist_raw', 'raw_category_translation') }}
),
joined as (
    select
        p.product_id,
        coalesce(t.product_category_name_english, 'uncategorized')  as category_en,
        p.product_category_name                                      as category_pt,
        p.product_name_length,
        p.product_description_length,
        p.product_photos_qty,
        p.product_weight_g,
        p.product_length_cm,
        p.product_height_cm,
        p.product_width_cm,
        -- volumetric weight in grams (L*H*W / 5000 * 1000)
        round(
            (p.product_length_cm * p.product_height_cm * p.product_width_cm) / 5.0,
            2
        )                                                            as volumetric_weight_g
    from products p
    left join translations t
        on p.product_category_name = t.product_category_name
    where p.product_id is not null
)
select * from joined
```

Việc dịch tên danh mục sản phẩm sang tiếng Anh (`category_en`) được thực hiện một lần duy nhất tại `stg_products` bằng `left join`, kèm giá trị mặc định `'uncategorized'` cho các danh mục chưa có bản dịch tương ứng - nhờ vậy, `category_revenue_monthly` ở mục 5.4.2 chỉ cần `ref()` tới model này mà không phải tự JOIN lại bảng tra cứu dịch thuật.

Toàn bộ 7 Staging Model được cấu hình vật chất hóa dưới dạng **view** (`+materialized: view` trong `dbt_project.yml`, trình bày chi tiết ở mục 5.5), thay vì bảng vật lý. Lựa chọn này phù hợp vì logic biến đổi ở tầng Staging chỉ gồm các phép ép kiểu, đổi tên cột và lọc dòng - chi phí tính toán thấp - trong khi lợi ích của việc dùng view là dữ liệu luôn phản ánh đúng trạng thái mới nhất của bảng thô mà không tốn thêm dung lượng lưu trữ trùng lặp. Đánh đổi tương ứng là mỗi truy vấn xuống các Data Mart phải tính toán lại logic biến đổi này, nhưng ở quy mô dữ liệu của dự án (hơn 100.000 đơn hàng), chi phí này không đáng kể so với lợi ích về tính đơn giản và tính mới của dữ liệu.

Mỗi Staging Model đều đi kèm một số ràng buộc kiểm tra chất lượng cơ bản khai báo trong `schema.yml`, ví dụ `order_id` của `stg_orders` phải `unique` và `not_null`, hay `review_score` của `stg_order_reviews` chỉ được nhận giá trị trong tập `{1, 2, 3, 4, 5}`. Các ràng buộc này là bước kiểm tra đầu tiên trong chuỗi Data Quality Testing của dự án; toàn bộ danh sách kiểm tra cùng cách chúng được thực thi tự động trong Airflow được trình bày đầy đủ ở mục 5.4.3.

![Cây thư mục dbt_project/models/staging/](image.png)
![Kết quả truy vấn `select * from staging.stg_orders limit 10;`](image-2.png)
