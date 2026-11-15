# databricks-ecommerce-medallion

# Databricks E-Commerce Analytics: End-to-End Medallion Architecture

An end-to-end data engineering pipeline built on **Databricks** and **Unity Catalog** implementing the **Medallion Architecture (Bronze → Silver → Gold)** for e-commerce sales analytics.

---

## Architecture Overview

```text
┌───────────────────────────┐
│   Raw CSV Data Feed       │
│  (5,000 E-Commerce Rows)  │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│   Unity Catalog Volume    │
│  ecommerce_analytics.     │
│  default.data_feed        │
└─────────────┬─────────────┘
              │  Idempotent Ingestion (COPY INTO / PySpark)
              │  + Audit Metadata (_ingested_at, _source_file)
              ▼
┌───────────────────────────┐
│      BRONZE LAYER         │
│  ecommerce_analytics.     │
│  bronze.ecommerce_sales   │
└─────────────┬─────────────┘
              │  Data Cleansing, Deduplication & Type Casting
              ▼
┌───────────────────────────┐
│      SILVER LAYER         │
│  ecommerce_analytics.     │
│  silver.*                 │
└─────────────┬─────────────┘
              │  Business Aggregations & KPI Modeling
              ▼
┌───────────────────────────┐
│       GOLD LAYER          │
│  ecommerce_analytics.     │
│  gold.*                   │
└───────────────────────────┘
```

---

## Tech Stack & Databricks Features
* **Platform:** Databricks Lakehouse Platform
* **Governance & Storage:** Unity Catalog (3-Level Namespace: `catalog.schema.table` & Managed Volumes)
* **Table Format:** Delta Lake (ACID transactions, schema evolution via `mergeSchema`, time travel)
* **Languages:** Databricks SQL & PySpark
* **Ingestion:** `COPY INTO` with idempotent file tracking & `_metadata.file_path` lineage

---

## Dataset Schema

The raw ingestion dataset (`ecommerce_sales_analytics_5000.csv`) contains **5,000 e-commerce order transactions** across four regions (`North`, `South`, `East`, `West`) and four product categories (`Beauty`, `Clothing`, `Electronics`, `Home`):

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `order_id` | `INT` | Unique identifier for each e-commerce order (e.g., `10001`–`15000`) |
| `order_date` | `DATE` | Date the order was placed |
| `customer_id` | `INT` | Unique identifier for the customer |
| `product_category` | `STRING` | Product category (`Beauty`, `Clothing`, `Electronics`, `Home`) |
| `region` | `STRING` | Geographic sales region (`North`, `South`, `East`, `West`) |
| `quantity` | `INT` | Number of units purchased in the order |
| `unit_price` | `DOUBLE` | Price per unit before discount |
| `discount` | `DOUBLE` | Discount rate applied to the order (e.g., `0.00` to `0.35`) |
| `payment_method` | `STRING` | Payment mode used (`Card`, `Wallet`, `COD`) |
| `delivery_days` | `INT` | Number of days taken to deliver the order |
| `customer_rating` | `DOUBLE` | Post-delivery customer rating (`1.0` to `5.0`) |
| `revenue` | `DOUBLE` | Net revenue generated after applying discount |

---

## Pipeline Implementation Details

### 1. Landing Layer (Unity Catalog Volume)
A managed Unity Catalog Volume is provisioned to govern non-tabular raw file ingestion prior to table creation:
```sql
CREATE VOLUME IF NOT EXISTS ecommerce_analytics.default.data_feed
COMMENT 'Managed volume for raw file ingestion';
```

### 2. Bronze Layer (Raw Ingestion & Lineage)
Raw CSV files are ingested from `/Volumes/ecommerce_analytics/default/data_feed/` into a managed Delta table (`ecommerce_analytics.bronze.ecommerce_sales`) using `COPY INTO`:
* **Idempotency:** `COPY INTO` tracks previously ingested files to prevent duplicate loads on pipeline re-runs.
* **Schema Evolution:** Enabled via `'mergeSchema' = 'true'`.
* **Audit Columns:** Enriched with `current_timestamp() AS _ingested_at` and `_metadata.file_path AS _source_file` for full data lineage.

### 3. Silver Layer (Cleansed & Conformed Entities)
The Silver schema (`ecommerce_analytics.silver`) transforms raw Bronze records into validated, analytics-ready tables:
* Deduplication on primary keys (`order_id`) using window functions (`ROW_NUMBER()`).
* Strict data type casting, null filtering, and standardization of categorical values (`region`, `product_category`, `payment_method`).
* Addition of processing audit timestamps (`_silver_processed_at`).

### 4. Gold Layer (Business-Level Aggregations)
The Gold schema (`ecommerce_analytics.gold`) provides curated, aggregated tables optimized for BI dashboards and executive reporting:
* **Regional Sales Performance:** Total revenue, units sold, order volume, and average discount by `region`.
* **Category & Payment Insights:** Revenue breakdown by `product_category` and `payment_method`, along with delivery SLA (`delivery_days`) and `customer_rating` metrics.
* **Freshness Tracking:** Includes `last_updated_at` timestamps for downstream dashboard transparency.

---

## How to Run This Project

1. **Prerequisites:** A Unity Catalog-enabled Databricks workspace (Databricks Runtime 13.3 LTS or above) with `CREATE CATALOG`, `CREATE SCHEMA`, and `CREATE VOLUME` privileges.
2. **Import Notebook:** Clone this repository into Databricks using **Git folders** (`Workspace > Create > Git folder`) or import the `.ipynb` notebook directly.
3. **Upload Raw Data:** Upload `ecommerce_sales_analytics_5000.csv` to `/Volumes/ecommerce_analytics/default/data_feed/` via **Catalog Explorer** or the notebook **File > Upload data to volume** menu.
4. **Execute Pipeline:** Run the notebook cells sequentially from Volume creation through Bronze, Silver, and Gold transformations.
