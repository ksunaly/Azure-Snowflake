# 📁 Week 2: Building an Incremental ETL Pipeline (Azure SQL → Snowflake)

## 🎯 Goals

- Prepare for **incremental data ingestion** from Azure SQL into Snowflake.
- Build staging (`TMP`) and final tables with **ETL audit columns**.
- Create and manage a **WATERMARK table** to track changes.
- Develop **idempotent ETL logic** using ADF and Snowflake **Stored Procedures**.
- Lay the foundation for a reusable, maintainable pipeline architecture.

---

### 1. Snowflake Table Creation

For each source table (`orders`, `customers`, `products`), create:

- **TMP tables**: Always truncated and reloaded with fresh data.
- **Final tables**: Persistent, clean data with audit metadata.
  
```sql
CREATE OR REPLACE TABLE raw.azure_sql.tmp_orders (
    order_id INT,
    customer_id INT,
    product_id INT,
    quantity INT,
    total_price DECIMAL(10,2),
    created_at DATETIME,
    updated_at DATETIME
);

CREATE OR REPLACE TABLE raw.azure_sql.orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    quantity INT,
    total_price DECIMAL(10,2),
    created_at DATETIME,
    updated_at DATETIME,
    etl_timestamp_utc TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);

CREATE OR REPLACE TABLE raw.azure_sql.tmp_customers (
    customer_id INT, 
    name VARCHAR(100),
    email VARCHAR(100),
    created_at DATETIME 
);

CREATE OR REPLACE TABLE raw.azure_sql.customers (
  customer_id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100),
  created_at DATETIME,
  etl_timestamp_utc TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);

CREATE OR REPLACE TABLE raw.azure_sql.tmp_products (
  product_id INT,
  name VARCHAR(100),
  category VARCHAR(50),
  price DECIMAL(10,2),
  created_at DATETIME
);


CREATE OR REPLACE TABLE raw.azure_sql.products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  category VARCHAR(50),
  price DECIMAL(10,2),
  created_at DATETIME,
  etl_timestamp_utc TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### 2.  Create WATERMARK table and CONFIG schema

Used to track incremental data loads across pipelines.

```sql
CREATE SCHEMA IF NOT EXISTS raw.config;

CREATE OR REPLACE TABLE raw.config.watermark (
    pipeline_name STRING,
    schema_name STRING,
    table_name STRING,
    last_created_at TIMESTAMP_NTZ,
    last_updated_at TIMESTAMP_NTZ,
    pipeline_start_ts TIMESTAMP_NTZ,
    pipeline_end_ts TIMESTAMP_NTZ,
    source STRING,
    target STRING
);

```

Insert procedure to automate watermark updates after each ETL run. Keeps history clean and trackable.

```sql
CREATE OR REPLACE PROCEDURE raw.config.sp_insert_watermark(
    p_pipeline_name STRING,
    p_schema_name STRING,
    p_table_name STRING,
    p_last_created_at TIMESTAMP_NTZ,
    p_last_updated_at TIMESTAMP_NTZ,
    p_pipeline_start_ts TIMESTAMP_NTZ,
    p_pipeline_end_ts TIMESTAMP_NTZ,
    p_source STRING,
    p_target STRING
)
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    INSERT INTO raw.config.watermark (
        pipeline_name,
        schema_name,
        table_name,
        last_created_at,
        last_updated_at,
        pipeline_start_ts,
        pipeline_end_ts,
        source,
        target
    )
    VALUES (
        p_pipeline_name,
        p_schema_name,
        p_table_name,
        p_last_created_at,
        p_last_updated_at,
        p_pipeline_start_ts,
        p_pipeline_end_ts,
        p_source,
        p_target
    );

    RETURN 'WATERMARK INSERTED SUCCESSFULLY';
END;
$$;

```


### 3.  Snowflake Stored Procedure

Now we need to extact only the newest queries, and save result in TMP table, 
```sql
CREATE OR REPLACE PROCEDURE raw.azure_sql.sp_merge_orders()
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    MERGE INTO raw.azure_sql.orders AS target
    USING raw.azure_sql.tmp_orders AS source
    ON target.order_id = source.order_id
    WHEN MATCHED THEN
        UPDATE SET 
            customer_id = source.customer_id,
            product_id = source.product_id,
            quantity = source.quantity,
            total_price = source.total_price,
            created_at = source.created_at,
            updated_at = source.updated_at,
            etl_timestamp_utc = CURRENT_TIMESTAMP()
    WHEN NOT MATCHED THEN
        INSERT (
            order_id,
            customer_id,
            product_id,
            quantity,
            total_price,
            created_at,
            updated_at,
            etl_timestamp_utc
        )
        VALUES (
            source.order_id,
            source.customer_id,
            source.product_id,
            source.quantity,
            source.total_price,
            source.created_at,
            source.updated_at,
            CURRENT_TIMESTAMP()
        );

    RETURN 'MERGE SUCCESSFUL';
END;
$$;

CALL raw.azure_sql.sp_merge_orders()
```

Repeat the same for other tables

### 4.  Debug pipeline

1. Make sure new tables are empty.
2. Let's make a first record for orders table in watermark table:

```sql
INSERT INTO raw.config.watermark (
    pipeline_name,
    schema_name,
    table_name,
    last_created_at,
    last_updated_at,
    pipeline_start_ts,
    pipeline_end_ts,
    source,
    target
)
VALUES (
    'orders_pipeline',
    'raw.azure_sql',
    'orders',
    '2000-01-15 00:00:00', -- Example last_created_at
    '2000-01-18 00:00:00', -- Example last_updated_at
    CURRENT_TIMESTAMP(),   -- pipeline_start_ts (can also use exact timestamp if needed)
    CURRENT_TIMESTAMP(),   -- pipeline_end_ts
    'azure_sql',
    'snowflake'
);

```
3. Run the pipeline in Azure
4. Add more records in Azure SQL and then run pipeline again
5. It should add new values in Snowflake table and update watermark table
   ![image](https://github.com/user-attachments/assets/ca121d55-af29-4834-9906-b382b25d5e54)

### 5. Schedule pipeline

Navigate to Data Factory → Monitoring → Diagnostic Settings.
Create a new Diagnostic Setting and send logs to Azure Monitor or Log Analytics.
Ensure you enable PipelineRuns and ActivityRuns logs.

It will start to send logs into Log Analytics, and we can query logs and create our own rules and dashboards for monitoring of pipelines.

Create an Alert Rule

Go to Azure Portal → Monitor → Alerts → + Create Alert Rule.

Resource: Choose your Data Factory instance.

![image](https://github.com/user-attachments/assets/5f2b6a9e-500a-4b0a-bcd6-013cadd69c53)

Condition:

Select Signal Type → Metric or Log.
Example: Use metric Pipeline failed runs or query where PipelineRunStatus == 'Failed'.
Action Group:

Create or select an existing Action Group.
Add notification channels (Email, SMS, Webhook, etc.).
You can configure alerts per pipeline.

Assign Severity levels (Sev 0 - Sev 4).

Integrate with ITSM systems (PagerDuty, ServiceNow, Slack) using Webhooks.







