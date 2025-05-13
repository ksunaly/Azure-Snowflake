## 📅 Week 1: Initial Setup and Data Ingestion from Azure SQL to Snowflake

### 🎯 Objectives

- Set up the foundational infrastructure on Azure and Snowflake.
- Create and populate source tables in Azure SQL Database.
- Configure Azure Data Factory (ADF) to perform a COPY activity from Azure SQL to Snowflake via Blob Storage.
- Perform an initial **full load** of data into the `RAW.AZURE_SQL.ORDERS` table in Snowflake.

---

### 1. 🔧 Azure Environment Setup

#### Azure Resource Group

Created a dedicated **Resource Group** via Azure Portal:

- **Portal path**: Azure Portal → “Resource Groups” → “+ Create”
- **Name**: `azure_env_wus2`
- **Region**: Same as planned for ADF and Storage

#### Azure SQL Server & Database

**SQL Server**

- **Portal path**: Azure Portal → “SQL servers” → “+ Create”
- **Server name**: `azure-sql-dev-wus2`
- **Admin login**: `sqladmin`
- **Password**: _[your password]_
- **Allow Azure services access**: Enabled

**SQL Database**

- **Name**: `azure-db-dev-wus2`
- **Pricing Tier**: Basic
- **Location**: Same as Resource Group

#### Azure Blob Storage

- **Name**: `azureblobdevwus2`

![Azure Blob Storage](https://github.com/user-attachments/assets/c771761b-fce9-4f6d-8085-814b51e2f43e)

#### Azure Data Factory

- **ADF Name**: `factorydata12357`
- **Location**: Same as other resources

---

### 2. ✅ Snowflake Setup

- Registered via [Snowflake Trial](https://signup.snowflake.com/)
- Logged into **Snowflake Web UI**
- Opened SQL worksheet and executed the following:

```sql
CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH;
CREATE DATABASE IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS RAW.azure_sql;
```

```sql
CREATE TABLE customers (
  customer_id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100),
  created_at DATETIME
);
```
---

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  category VARCHAR(50),
  price DECIMAL(10,2),
  created_at DATETIME
);
```

```sql
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  customer_id INT,
  product_id INT,
  quantity INT,
  total_price DECIMAL(10,2),
  created_at DATETIME,
  updated_at DATETIME
);
```

### 3. ✅ Creating Tables in Azure SQL

In Azure SQL Database:

![SQL Tables](https://github.com/user-attachments/assets/986b4e65-bd63-4a58-b017-0ce1b87b7b5b)

#### Run the following scripts:

```sql
CREATE TABLE customers (
  customer_id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100),
  created_at DATETIME
);
```

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  category VARCHAR(50),
  price DECIMAL(10,2),
  created_at DATETIME
);
```

```sql
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  customer_id INT,
  product_id INT,
  quantity INT,
  total_price DECIMAL(10,2),
  created_at DATETIME,
  updated_at DATETIME
);
```

#### Insert test data:

```sql
INSERT INTO customers (customer_id, name, email, created_at) VALUES
(1, 'Alice Johnson', 'alice.j@example.com', '2023-01-10 08:23:45'),
(2, 'Bob Smith', 'bob.smith@example.com', '2023-01-12 14:11:09'),
(3, 'Charlie Davis', 'charlie.d@example.com', '2023-01-15 09:37:22'),
(4, 'Diana Cruz', 'diana.c@example.com', '2023-02-01 17:02:18'),
(5, 'Ethan Lee', 'ethan.l@example.com', '2023-02-10 11:45:30');

INSERT INTO products (product_id, name, category, price, created_at) VALUES
(101, 'Laptop Pro 15"', 'Electronics', 1499.99, '2023-01-01 10:05:10'),
(102, 'Wireless Mouse', 'Accessories', 29.99, '2023-01-03 15:25:40'),
(103, 'Noise Cancelling Headphones', 'Electronics', 199.95, '2023-01-05 12:14:50'),
(104, 'Office Desk', 'Furniture', 299.99, '2023-01-08 09:09:09'),
(105, 'LED Monitor 24"', 'Electronics', 159.99, '2023-01-09 18:30:00');

INSERT INTO orders (order_id, customer_id, product_id, quantity, total_price, created_at, updated_at) VALUES
(1001, 1, 101, 1, 1499.99, '2023-01-11 10:15:23', '2023-01-11 12:45:30'),
(1002, 2, 102, 2, 59.98, '2023-01-13 08:20:40', '2023-01-15 09:10:15'),
(1003, 3, 103, 1, 199.95, '2023-01-16 14:50:10', '2023-01-18 16:22:55'),
(1004, 4, 104, 1, 299.99, '2023-02-02 13:14:00', '2023-02-02 13:14:00'),
(1005, 5, 105, 2, 319.98, '2023-02-12 11:11:11', '2023-02-14 15:01:20'),
(1006, 1, 102, 3, 89.97, '2023-03-01 07:07:07', '2023-03-04 08:08:08');
```

---

### 4. ✅ Prepare ADF

#### Linked Services

##### Azure SQL DB

- Go to **Manage → Linked Services → New**
- Choose **Azure SQL DB**

![Azure SQL DB](https://github.com/user-attachments/assets/9dec97c5-fccc-4a80-8cd3-f16d63b0c6e3)

Fill in the connection details:

![SQL DB Setup](https://github.com/user-attachments/assets/11fb4d5b-ff30-4129-b09e-87423cfc967e)

##### Azure Blob Storage

- Go to **Manage → Linked Services → New**
- Choose **Azure Blob Storage**

![Blob Storage](https://github.com/user-attachments/assets/56cb0724-5914-4e6d-a99d-d05e7488af44)

##### Snowflake

- Go to **Manage → Linked Services → New**
- Choose **Snowflake**


#### Create Pipeline

- In ADF, go to **Author → Add Resource → Pipeline**
- Rename the pipeline
- Add **Copy Data** activity:

![Copy Data](https://github.com/user-attachments/assets/90baebc7-5985-4e9d-90f5-b0e850b501c6)

- Name the activity
- Go to **Source** tab → Click "+" to add a new dataset → Select **SQL DB**

![SQL Source](https://github.com/user-attachments/assets/92097681-0b77-48e8-b294-0f4d8357d603)

- Fill in source properties:

![Source Properties](https://github.com/user-attachments/assets/99c3e2aa-1299-4f13-8ad8-59888f6fdc2a)

- Go to **Sink** → Add new → Select **Snowflake**

![Snowflake Sink](https://github.com/user-attachments/assets/057eaa9b-a3dc-4109-826c-0fae1d8d3323)

Fill in sink properties
![image](https://github.com/user-attachments/assets/9887e0a2-faf9-4cd5-b428-a76060514cf9)


- Mapping: click **Import Schemas**
- In **Settings**, choose staging location as **Azure Blob Storage** and define path:

![Staging Settings](https://github.com/user-attachments/assets/6a099e3d-1103-4676-bc2e-89bed140f056)

- Click Validate and Debug 
---

### ✅ Validate in Snowflake

Run the following SQL to confirm data is uploaded:

```sql
SELECT * FROM RAW.azure_sql.products;
SELECT * FROM RAW.azure_sql.customers;
SELECT * FROM RAW.azure_sql.orders;
```

![Snowflake Result](https://github.com/user-attachments/assets/3426f0a2-9043-470b-a831-f089fa2acc20)
