````markdown
---
title: "5.4.1 - Preparing the Staging Layer (Ingestion & Staging Models)"
date: 2026-07-23
weight: 1
chapter: false
pre: " <b> 5.4.1 </b> "
---

This section describes two sequential processes. First, the mechanism for loading raw data from Amazon S3 into the `staging` schema on Amazon Redshift. Second, how dbt Core standardizes these raw tables into a collection of **Staging Models**—the single intermediate layer that all Data Mart models in Section 5.4.2 reference through the `ref()` function. Separating these two steps (loading raw data unchanged, then cleaning and transforming it with SQL) reflects the essence of the ELT architecture introduced in Section 5.1: all data transformations are performed inside Redshift rather than in an external processing layer.

## 1. Loading Raw Data into Redshift Using the COPY Command

The data loading process is not performed manually. Instead, it is handled by the `ingestion/load_to_redshift.py` script, which is executed by the Airflow task `load_staging_to_redshift` (described in detail in Section 5.4.3). Rather than reading CSV files and inserting rows one by one, the script leverages Redshift's `COPY` command—the most efficient bulk-loading mechanism available on the platform. `COPY` takes advantage of Redshift's Massively Parallel Processing (MPP) architecture by allowing multiple compute nodes to read data directly from Amazon S3 in parallel.

All eight raw tables are defined centrally as a list of `TableSpec` objects (a Python dataclass). Each object specifies the source CSV file, destination table name, column definitions, the `CREATE TABLE` statement, and optional distribution (`DISTKEY`) and sort (`SORTKEY`) keys optimized for Redshift. This design makes adding a new source table straightforward: only a new `TableSpec` definition is required, without modifying the loading logic itself.

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
    # ... Remaining 7 TableSpec objects:
    #     raw_customers, raw_order_items,
    #     raw_order_payments, raw_order_reviews,
    #     raw_products, raw_sellers,
    #     raw_category_translation
]
````

The following table summarizes all eight raw tables created and loaded into the `staging` schema:

| Destination Table          | Source CSV File                         | Distribution Strategy  | Notes                                                               |
| :------------------------- | :-------------------------------------- | :--------------------- | :------------------------------------------------------------------ |
| `raw_orders`               | `olist_orders_dataset.csv`              | `DISTKEY(order_id)`    | Sorted by `order_purchase_timestamp` to optimize time-based queries |
| `raw_order_items`          | `olist_order_items_dataset.csv`         | `DISTKEY(order_id)`    | One record per order item                                           |
| `raw_customers`            | `olist_customers_dataset.csv`           | `DISTKEY(customer_id)` | Both distributed and sorted by `customer_id`                        |
| `raw_sellers`              | `olist_sellers_dataset.csv`             | `DISTSTYLE ALL`        | Small dimension table replicated across all nodes for faster joins  |
| `raw_products`             | `olist_products_dataset.csv`            | `DISTSTYLE ALL`        | Small dimension table replicated across all nodes                   |
| `raw_order_payments`       | `olist_order_payments_dataset.csv`      | `DISTKEY(order_id)`    | An order may contain multiple payment records (e.g., installments)  |
| `raw_order_reviews`        | `olist_order_reviews_dataset.csv`       | `DISTKEY(order_id)`    |                                                                     |
| `raw_category_translation` | `product_category_name_translation.csv` | `DISTSTYLE ALL`        | Lookup table used to translate product category names               |

Each table includes an additional `_loaded_at TIMESTAMP DEFAULT GETDATE()` column. This column is referenced as the `loaded_at_field` in `sources.yml`, enabling dbt to perform source freshness checks with a warning threshold of 25 hours and an error threshold of 49 hours after the most recent load. These thresholds are intentionally set slightly longer than the daily pipeline schedule, allowing a delayed execution without immediately flagging the source as stale.

One notable design decision is that the above list **does not include** the geolocation dataset (`olist_geolocation_dataset.csv`), even though this file is still downloaded and uploaded to Amazon S3 by the `ingestion/ingest_olist_to_s3.py` script. This omission is intentional rather than accidental. The geolocation dataset is relatively large but is rarely used directly by the Data Mart models described in Section 5.4.2. Therefore, it remains in the Bronze layer on Amazon S3 and is queried on demand through AWS Glue Crawler and Amazon Athena, as introduced in Section 5.1, instead of consuming additional Redshift storage and loading resources.

From a loading perspective, each table follows a **TRUNCATE followed by COPY** strategy (full refresh) rather than incremental loading:

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

Using a full-refresh approach (`TRUNCATE + COPY`) instead of incremental loading is appropriate for the Olist dataset, which is static and relatively small (approximately 100,000 orders). Reloading the entire dataset daily incurs minimal cost while significantly simplifying the ingestion logic and making the process easier to validate. The loading of all eight tables is wrapped within a single database transaction (`conn.autocommit = False`). If loading any table fails, `conn.rollback()` is executed, ensuring that the entire transaction is rolled back and preventing the `staging` schema from ending up in a partially updated state.

For authentication, the `_auth_clause()` function prioritizes using an `IAM_ROLE`—the same IAM Role provisioned by Terraform and attached to the Redshift Namespace in Section 5.3.1. Static AWS Access Key and Secret Key credentials are used only as a fallback option. This approach aligns with the principle of least privilege discussed in Section 5.2: because the `COPY` command executes inside Redshift, it can assume temporary permissions through the attached IAM Role without embedding long-lived credentials.

![Running load\_to\_redshift.py](image-1.png)
````markdown
## 2. Defining the dbt Staging Models

Once the raw data has been loaded into the `staging` schema, dbt takes over the remainder of the Silver layer by standardizing data types, renaming columns according to consistent naming conventions, filtering invalid records, and deriving additional fields that are directly consumed by the Data Mart models. The project defines **seven Staging Models**, each corresponding to a raw table, located under `models/staging/` and following the naming convention `stg_<entity_name>`.

| Model | Source (`source`) | Main Transformation Logic |
| :--- | :--- | :--- |
| `stg_customers` | `raw_customers` | Standardizes postal codes to 8 characters using `lpad`, capitalizes city names, and converts state codes to uppercase |
| `stg_sellers` | `raw_sellers` | Applies the same address normalization rules as `stg_customers` |
| `stg_order_items` | `raw_order_items` | Casts price and freight values to `decimal(10,2)`, calculates `total_item_revenue`, and removes rows with negative values |
| `stg_order_payments` | `raw_order_payments` | Normalizes `payment_type` to lowercase and filters out records with negative `payment_value` |
| `stg_order_reviews` | `raw_order_reviews` | Classifies review scores into `positive`, `neutral`, and `negative`, and calculates `days_to_answer` |
| `stg_products` | `raw_products`, `raw_category_translation` | Joins with the category translation table to obtain English product categories and calculates `volumetric_weight_g` |
| `stg_orders` | `raw_orders` | Standardizes order status, derives the `is_delivered` and `is_late_delivery` flags, and calculates delivery delay in days |

Among these models, `stg_orders` contains the most complex business logic. Most order lifecycle calculations—which are reused across multiple Data Mart models described in Section 5.4.2—are computed once here rather than being repeatedly implemented in each Gold-layer model.

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

The three derived columns—`order_date`, `order_week`, and `order_month`—are precomputed here so that every Data Mart requiring time-based aggregation (such as `revenue_daily`, `category_revenue_monthly`, and `fct_customer_cohorts`) shares the same definition of the order date. This prevents individual models from implementing their own `date_trunc` logic differently. Likewise, the `is_delivered` and `is_late_delivery` flags are defined explicitly and centrally within this model, rather than requiring each downstream Data Mart to reimplement the comparison between the actual and estimated delivery dates.

`stg_products` demonstrates another common transformation pattern: combining multiple data sources within the Staging layer to produce a single, analysis-ready dataset for downstream models.

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

        -- volumetric weight in grams (L × H × W / 5000 × 1000)
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

The translation of product category names into English (`category_en`) is performed once within `stg_products` using a `LEFT JOIN`, with `'uncategorized'` assigned as the default value for categories without a matching translation. Consequently, the `category_revenue_monthly` model discussed in Section 5.4.2 only needs to reference this staging model through `ref()` instead of repeatedly joining the translation lookup table.

All seven Staging Models are materialized as **views** (`+materialized: view` in `dbt_project.yml`, described in detail in Section 5.5) rather than physical tables. This choice is appropriate because the transformations in the Staging layer primarily consist of type casting, column renaming, and record filtering, all of which incur relatively low computational overhead. The main advantage of using views is that they always reflect the latest state of the underlying raw tables without consuming additional storage for duplicated data. The corresponding trade-off is that downstream Data Mart queries must recompute these lightweight transformations at runtime. However, given the project's scale (approximately 100,000 orders), this overhead is negligible compared with the benefits of maintaining a simpler architecture and ensuring data freshness.

Each Staging Model is accompanied by a set of basic data quality constraints defined in `schema.yml`. For example, the `order_id` column in `stg_orders` is tested for both `unique` and `not_null`, while the `review_score` column in `stg_order_reviews` is validated to ensure its value belongs to the set `{1, 2, 3, 4, 5}`. These tests constitute the first layer of the project's Data Quality Testing framework. The complete set of tests and their automated execution within Airflow are described in detail in Section 5.4.3.

![dbt_project/models/staging Directory Structure](image.png)

![Query Result of `select * from staging.stg_orders limit 10;`](image-2.png)

