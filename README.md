# E-Commerce Data Pipeline

## Project Overview

This project demonstrates a modern ELT (Extract, Load, Transform) data engineering pipeline using:

* Mock REST API
* Python
* Snowflake
* dbt
* Power BI

The pipeline extracts e-commerce order data from a REST API, loads raw JSON into Snowflake, transforms the data using dbt, and visualizes analytics using Power BI.

---

# Architecture

```text
Mock API → Python → Snowflake → dbt → Power BI
```

---

# Folder Structure

```text
E-commerce-Data-Pipeline/
│
├── scripts/
│   ├── fetch_orders.py
│   ├── snowflake_loader.py
│   └── requirements.txt
│
├── dbt_project/
│   ├── dbt_project.yml
│   ├── profiles.yml
│   │
│   └── models/
│       ├── staging/
│       │   ├── stg_orders.sql
│       │   └── stg_order_items.sql
│       │
│       └── marts/
│           ├── fact_orders.sql
│           ├── dim_products.sql
│           └── dim_customers.sql
│
├── snowflake/
│   ├── init_setup.sql
│   └── create_tables.sql
│
├── powerbi/
│   └── powerbi_setup.md
│
├── .env
│
└── README.md
```

---

# 1. Snowflake Setup

## init_setup.sql

```sql
CREATE DATABASE IF NOT EXISTS ecommerce_db;

CREATE SCHEMA IF NOT EXISTS ecommerce_db.raw;

CREATE SCHEMA IF NOT EXISTS ecommerce_db.analytics;
```

---

## create_tables.sql

```sql
CREATE TABLE IF NOT EXISTS ecommerce_db.raw.orders (
    src_json VARIANT,
    loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);
```

---

# 2. Python Scripts

## requirements.txt

```text
requests
pandas
snowflake-connector-python
python-dotenv
```

---

## fetch_orders.py

```python
import requests
import json

url = "https://fakestoreapiserver.reactbd.org/api/orders?page=1&perPage=500"

response = requests.get(url)

data = response.json()

print(json.dumps(data, indent=2))
```

---

## snowflake_loader.py

```python
import json
import requests
import snowflake.connector
from dotenv import load_dotenv
import os

load_dotenv()

conn = snowflake.connector.connect(
    user=os.getenv("SNOWFLAKE_USER"),
    password=os.getenv("SNOWFLAKE_PASSWORD"),
    account=os.getenv("SNOWFLAKE_ACCOUNT"),
    warehouse=os.getenv("SNOWFLAKE_WAREHOUSE"),
    database=os.getenv("SNOWFLAKE_DATABASE"),
    schema=os.getenv("SNOWFLAKE_SCHEMA")
)

cursor = conn.cursor()

url = "https://fakestoreapiserver.reactbd.org/api/orders?page=1&perPage=500"

response = requests.get(url)

data = response.json()

orders = data.get("data", [])

for order in orders:
    cursor.execute(
        """
        INSERT INTO ecommerce_db.raw.orders (src_json)
        SELECT PARSE_JSON(%s)
        """,
        (json.dumps(order),)
    )

print("Data loaded successfully")

cursor.close()
conn.close()
```

---

# 3. dbt Project

## dbt_project.yml

```yaml
name: 'ecommerce_dbt_project'
version: '1.0.0'
config-version: 2

profile: 'ecommerce_profile'

model-paths: ["models"]

models:
  ecommerce_dbt_project:
    staging:
      materialized: view
    marts:
      materialized: table
```

---

## profiles.yml

```yaml
ecommerce_profile:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: your_account
      user: your_user
      password: your_password
      role: accountadmin
      database: ecommerce_db
      warehouse: compute_wh
      schema: analytics
      threads: 1
```

---

## stg_orders.sql

```sql
WITH source_data AS (

    SELECT
        src_json,
        loaded_at
    FROM ecommerce_db.raw.orders

)

SELECT
    src_json:_id::INT                AS order_id,
    src_json:user:_id::INT           AS customer_id,
    src_json:status::STRING          AS order_status,
    src_json:createdAt::TIMESTAMP    AS order_date,
    src_json:total::FLOAT            AS total_amount,
    loaded_at

FROM source_data;
```

---

## stg_order_items.sql

```sql
WITH raw_data AS (

    SELECT
        src_json,
        loaded_at
    FROM ecommerce_db.raw.orders

),

flattened AS (

    SELECT
        src_json:_id::INT                    AS order_id,
        value:productId::INT                 AS product_id,
        value:name::STRING                   AS product_name,
        value:quantity::INT                  AS quantity,
        value:price::FLOAT                   AS price,
        loaded_at

    FROM raw_data,
    LATERAL FLATTEN(input => src_json:items)

)

SELECT * FROM flattened;
```

---

## fact_orders.sql

```sql
SELECT
    order_id,
    customer_id,
    total_amount,
    order_status,
    order_date
FROM {{ ref('stg_orders') }}
```

---

## dim_products.sql

```sql
SELECT DISTINCT
    product_id,
    product_name
FROM {{ ref('stg_order_items') }}
```

---

## dim_customers.sql

```sql
SELECT DISTINCT
    customer_id
FROM {{ ref('stg_orders') }}
```

---

# 4. Environment Variables

## .env

```env
SNOWFLAKE_USER=your_user
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_ACCOUNT=your_account
SNOWFLAKE_WAREHOUSE=compute_wh
SNOWFLAKE_DATABASE=ecommerce_db
SNOWFLAKE_SCHEMA=raw
```

---

# 5. Power BI Setup

## powerbi_setup.md

```text
1. Open Power BI Desktop
2. Click Get Data
3. Select Snowflake
4. Enter Snowflake credentials
5. Select analytics tables
6. Build dashboards
```

---

# How to Run

## Step 1

Run Snowflake setup scripts.

## Step 2

Install dependencies.

```bash
pip install -r requirements.txt
```

## Step 3

Run Python loader.

```bash
python snowflake_loader.py
```

## Step 4

Run dbt.

```bash
dbt debug
dbt run
dbt test
```

## Step 5

Connect Power BI.

---

# Dashboard Ideas

* Revenue Analysis
* Order Trends
* Product Performance
* Customer Insights
* Order Status Analytics

---

# Resume Description

Built an end-to-end ELT data pipeline using Python, Snowflake, dbt, and Power BI for ingesting and transforming e-commerce API data into analytics-ready dashboards.

---

# Tech Stack

| Layer          | Technology |
| -------------- | ---------- |
| Source         | Mock API   |
| Ingestion      | Python     |
| Warehouse      | Snowflake  |
| Transformation | dbt        |
| Visualization  | Power BI   |

---

# Future Improvements

* Add Apache Airflow orchestration
* Add incremental dbt models
* Add CI/CD pipeline
* Add Docker support
* Add data quality tests
