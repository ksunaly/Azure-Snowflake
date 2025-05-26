# 📁 Week 2: Building an Incremental ETL Pipeline (Azure SQL → Snowflake)

## 🎯 Goals

- Prepare for **incremental data ingestion** from Azure SQL into Snowflake.
- Build staging (`TMP`) and final tables with **ETL audit columns**.
- Create and manage a **WATERMARK table** to track changes.
- Develop **idempotent ETL logic** using ADF and Snowflake **Stored Procedures**.
- Lay the foundation for a reusable, maintainable pipeline architecture.

---

### Architecture

![image](https://github.com/user-attachments/assets/6ff78d48-8c05-41fe-a4b7-00353ad9b4a2)


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

1. Go to Azure Data Studio
2. Click on Author
3. Under pipelines select 'PL_CopyOrders_SQL_to_Snowflake' pipeline that was created before
4. Click on Add Trigger - New/Edit ![image](https://github.com/user-attachments/assets/58facdb2-dce6-4724-a371-cc9bd30ea941)
5. Choose nw trigger in pop out window
6. Select necessary setting, including when you want to start triggering pipelines, how often ![image](https://github.com/user-attachments/assets/fe3cabd5-2e68-4b13-8223-de784ecaa288)
7. Save
8. Publish

### 6. Add Alerts on Activity Failure

1. Go to Azure Portal → open your Data Factory instance.
2. In the left menu, under Monitoring, click Diagnostic settings.
3. Click + Add diagnostic setting.
4. Set the name, e.g., adf-logs.
5. Under Logs, check:
✅ PipelineRuns
✅ ActivityRuns
6. Under Destination, choose:
✅ Send to Log Analytics
7. Choose a Log Analytics workspace (create one if you don’t have it).
8. Click Save.
   ![image](https://github.com/user-attachments/assets/d8c1163a-9db8-464d-8893-384e18c9ace6)

→ ✅ Now your ADF will start sending logs to Log Analytics.

## 🚨 7. Create Alert for Pipeline Failures

- Go to **Azure Portal → Monitor → Alerts → + New alert rule**.
- **Scope**: Select your Azure Data Factory instance.
- **Condition**:
  - **Signal**: `Failed pipeline runs`
  - **Operator**: Greater than
  - **Threshold**: 0
  - **Aggregation**: Total
  - **Frequency**: Every 5 minutes
- **Action Group**:
  - Create or select one with **email notifications** (e.g., `EmailADFAlerts`)
- **Severity**: Choose `Sev 2` or appropriate level.
- Click **Create** to finalize the alert rule.

 ![image](https://github.com/user-attachments/assets/d09025ae-bc67-4f19-945c-133139523c80)

 ![image](https://github.com/user-attachments/assets/49734df9-192d-452e-80a4-ebc1720f79ef)













