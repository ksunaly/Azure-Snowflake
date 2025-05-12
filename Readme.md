## 📅 Week 1: Initial Setup and Data Ingestion from Azure SQL to Snowflake

### 🎯 Objectives

- Set up the foundational infrastructure on Azure and Snowflake.
- Create and populate source tables in Azure SQL Database.
- Configure Azure Data Factory (ADF) to perform a COPY activity from Azure SQL to Snowflake via Blob Storage.
- Perform an initial **full load** of data into the `RAW.AZURE_SQL.ORDERS` table in Snowflake.

---### 1. 🔧 Azure Environment Setup
Azure Resource Group

Created a dedicated **Resource Group** via Azure Portal:

- **Portal path**: Azure Portal → “Resource Groups” → “+ Create”
- **Name**: `azure_env_wus2`
- **Region**: Same region as planned for ADF and Storage

---

 Azure SQL Server & Database

**SQL Server**

- **Portal path**: Azure Portal → “SQL servers” → “+ Create”
- **Server name**: `azure-sql-dev-wus2`
- **Admin login**: `sqladmin`
- **Password**: 
- **Allow Azure services access**: Enabled

**SQL Database**

- **Name**: `azure-db-dev-wus2`
- **Pricing Tier**: Basic 
- **Location**: Same as Resource Group

 Azure Blob Storage 

- **Name**: azureblobdevwus2
![image](https://github.com/user-attachments/assets/c771761b-fce9-4f6d-8085-814b51e2f43e)

Azure Data Factory

**Created ADF Instance**
 **Name**: `factorydata12357`
 **Location**: Same as other resources

 ### 2. ✅ Snowflake Setup

 - Registered via [Snowflake Trial](https://signup.snowflake.com/)
 - Logged into **Snowflake Web UI**
 - Go to create- sql worksheet and paste code to create arehouse and database
```sql
CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH;
CREATE DATABASE IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS RAW.azure_sql;

### 3. ✅ Creating Tables in Azure SQL

In Azure SQL Database ![image](https://github.com/user-attachments/assets/986b4e65-bd63-4a58-b017-0ce1b87b7b5b)
 Ran the following SQL scripts:
```sql
  CREATE TABLE customers (
  customer_id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100),
  created_at DATETIME
);

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100),
  category VARCHAR(50),
  price DECIMAL(10,2),
  created_at DATETIME
);

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
--Now insert values in these table, should be different timestamps
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

### 4. ✅ Prepare ADF

In Azure Data Factory, configure the following linked services:
Azure SQL DB

go to manage - linked services -new  in Azure Data Factory ![image](https://github.com/user-attachments/assets/9dec97c5-fccc-4a80-8cd3-f16d63b0c6e3)
select Azure SQL DB
and fill in ![image](https://github.com/user-attachments/assets/11fb4d5b-ff30-4129-b09e-87423cfc967e)

Azure Blob Storage

![image](https://github.com/user-attachments/assets/56cb0724-5914-4e6d-a99d-d05e7488af44)

Snowflake

in Data Factory , go to Author - Add new resourse - Pipeline
rename it and in activites find move copy data: ![image](https://github.com/user-attachments/assets/90baebc7-5985-4e9d-90f5-b0e850b501c6)






