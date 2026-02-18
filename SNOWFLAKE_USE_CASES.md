# Snowflake: Zero to Advanced Level Use Cases

A comprehensive guide covering 50 real-world Snowflake use cases, from beginner to advanced, to help you master data warehousing, analytics, and cloud data platforms.

---

## Table of Contents

1. [Beginner Level (1-10)](#beginner-level)
2. [Intermediate Level (11-25)](#intermediate-level)
3. [Advanced Level (26-40)](#advanced-level)
4. [Expert Level (41-50)](#expert-level)

---

## Beginner Level

### 1. Creating Your First Database and Table
**Description:** Set up a basic database and create a simple table to store customer data.

```sql
CREATE DATABASE customer_db;
CREATE SCHEMA public;

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_date DATE
);
```

**Learning Outcome:** Understand Snowflake object hierarchy (Database → Schema → Table)

---

### 2. Loading Data from CSV Files
**Description:** Load sample CSV data into Snowflake using the COPY command.

```sql
CREATE STAGE my_stage
  URL = 's3://my-bucket/path/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');

COPY INTO customers
  FROM @my_stage/customers.csv
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
```

**Learning Outcome:** Master data ingestion and staging in Snowflake

---

### 3. Basic SELECT Queries
**Description:** Write simple queries to retrieve and filter data.

```sql
-- Retrieve all customers
SELECT * FROM customers;

-- Filter customers by creation date
SELECT name, email 
FROM customers 
WHERE created_date > '2025-01-01';
```

**Learning Outcome:** Master basic SQL querying in Snowflake

---

### 4. Using Snowflake Roles and Users
**Description:** Create users and roles with specific permissions.

```sql
CREATE USER analyst_user 
  PASSWORD = 'SecurePassword123!';

CREATE ROLE analyst_role;
GRANT SELECT ON DATABASE customer_db TO ROLE analyst_role;
GRANT ROLE analyst_role TO USER analyst_user;
```

**Learning Outcome:** Understand Snowflake security and access control

---

### 5. Aggregation and GROUP BY Operations
**Description:** Perform aggregations to summarize customer data.

```sql
SELECT 
    DATE_TRUNC('MONTH', created_date) AS month,
    COUNT(*) AS customer_count,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM customers
GROUP BY DATE_TRUNC('MONTH', created_date)
ORDER BY month DESC;
```

**Learning Outcome:** Master aggregation functions and grouping

---

### 6. Creating Views for Simplified Access
**Description:** Create views to provide simplified data access.

```sql
CREATE VIEW active_customers AS
SELECT customer_id, name, email
FROM customers
WHERE created_date >= DATEADD(YEAR, -1, CURRENT_DATE());
```

**Learning Outcome:** Learn view creation for code reusability

---

### 7. Working with Dates and Time Functions
**Description:** Master Snowflake date/time functions for temporal analysis.

```sql
SELECT 
    customer_id,
    created_date,
    DATEDIFF(DAY, created_date, CURRENT_DATE()) AS days_since_creation,
    YEAR(created_date) AS signup_year,
    MONTH(created_date) AS signup_month
FROM customers;
```

**Learning Outcome:** Understand Snowflake's date/time functions

---

### 8. Joining Multiple Tables
**Description:** Create and join related tables for comprehensive analysis.

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_amount DECIMAL(10, 2),
    order_date DATE
);

SELECT 
    c.name,
    COUNT(o.order_id) AS total_orders,
    SUM(o.order_amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

**Learning Outcome:** Master JOIN operations in Snowflake

---

### 9. Using String Functions
**Description:** Manipulate and transform string data.

```sql
SELECT 
    customer_id,
    UPPER(name) AS name_uppercase,
    LOWER(email) AS email_lowercase,
    LENGTH(name) AS name_length,
    SUBSTRING(email, 1, POSITION('@' IN email) - 1) AS username
FROM customers;
```

**Learning Outcome:** Learn Snowflake string manipulation functions

---

### 10. Understanding Snowflake Warehouses
**Description:** Create and manage compute warehouses for query execution.

```sql
CREATE WAREHOUSE analytics_wh
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 600
  AUTO_RESUME = TRUE;

-- Use the warehouse
USE WAREHOUSE analytics_wh;

-- Check warehouse details
SHOW WAREHOUSES;
```

**Learning Outcome:** Understand Snowflake's compute architecture

---

## Intermediate Level

### 11. Window Functions for Advanced Analytics
**Description:** Use window functions to perform row-by-row calculations.

```sql
SELECT 
    customer_id,
    order_date,
    order_amount,
    SUM(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) AS cumulative_amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id 
        ORDER BY order_date DESC
    ) AS order_rank
FROM orders;
```

**Learning Outcome:** Master window functions for complex analytics

---

### 12. Working with Semi-Structured Data (JSON)
**Description:** Query and parse JSON data in Snowflake.

```sql
CREATE TABLE events (
    event_id INT,
    event_data VARIANT
);

SELECT 
    event_id,
    event_data:event_type::STRING AS event_type,
    event_data:user_id::INT AS user_id,
    event_data:timestamp::TIMESTAMP AS event_time
FROM events;
```

**Learning Outcome:** Handle semi-structured data in Snowflake

---

### 13. Creating and Using Stored Procedures
**Description:** Build reusable logic with stored procedures.

```sql
CREATE PROCEDURE update_customer_status(customer_id INT, status VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    UPDATE customers 
    SET status = $2 
    WHERE customer_id = $1;
    RETURN 'Customer status updated successfully';
END;
$$;

CALL update_customer_status(1, 'ACTIVE');
```

**Learning Outcome:** Master procedural programming in Snowflake

---

### 14. User-Defined Functions (UDFs)
**Description:** Create custom functions for business logic.

```sql
CREATE OR REPLACE FUNCTION calculate_customer_lifetime_value(customer_id INT)
RETURNS DECIMAL(10, 2)
LANGUAGE SQL
AS
$$
    SELECT COALESCE(SUM(order_amount), 0)
    FROM orders
    WHERE customer_id = $1;
$$;

SELECT customer_id, calculate_customer_lifetime_value(customer_id) AS clv
FROM customers;
```

**Learning Outcome:** Create reusable business logic with UDFs

---

### 15. Implementing Slowly Changing Dimensions (SCD Type 2)
**Description:** Track historical changes in customer data.

```sql
CREATE TABLE customer_scd (
    customer_id INT,
    name VARCHAR,
    email VARCHAR,
    valid_from DATE,
    valid_to DATE,
    is_current BOOLEAN
);

-- Insert new version when customer info changes
INSERT INTO customer_scd (customer_id, name, email, valid_from, valid_to, is_current)
SELECT 
    c.customer_id,
    c.name,
    c.email,
    CURRENT_DATE(),
    '9999-12-31'::DATE,
    TRUE
FROM customers c;
```

**Learning Outcome:** Implement dimensional modeling patterns

---

### 16. Using Clustering Keys for Performance
**Description:** Optimize query performance with clustering keys.

```sql
CREATE TABLE sales_data (
    sales_id INT,
    region VARCHAR,
    product_id INT,
    sale_date DATE,
    amount DECIMAL(10, 2)
)
CLUSTER BY (region, sale_date);
```

**Learning Outcome:** Optimize data layout for query performance

---

### 17. Time Travel and Data Recovery
**Description:** Access historical versions of data using Time Travel.

```sql
-- Query data as it was 1 day ago
SELECT * FROM customers AT (OFFSET => -86400);

-- Query data at a specific point in time
SELECT * FROM customers BEFORE (TIMESTAMP => '2026-02-15 10:00:00');

-- Restore table to a previous state
UNDROP TABLE customers;
```

**Learning Outcome:** Use Snowflake's Time Travel feature for recovery

---

### 18. Implementing Incremental Loads with Merge
**Description:** Efficiently update tables with new data using MERGE.

```sql
MERGE INTO customers c
USING staging_customers s
ON c.customer_id = s.customer_id
WHEN MATCHED AND c.email != s.email 
    THEN UPDATE SET c.email = s.email, c.updated_date = CURRENT_DATE()
WHEN NOT MATCHED 
    THEN INSERT (customer_id, name, email, created_date) 
         VALUES (s.customer_id, s.name, s.email, CURRENT_DATE());
```

**Learning Outcome:** Master efficient incremental data loading

---

### 19. Creating Dynamic Queries with Variables
**Description:** Use variables for parameterized queries.

```sql
SET customer_threshold = 100;

SELECT 
    customer_id,
    name,
    COUNT(order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
HAVING COUNT(order_id) >= $customer_threshold;
```

**Learning Outcome:** Write flexible, reusable queries

---

### 20. Data Quality Monitoring with Automated Queries
**Description:** Implement data quality checks.

```sql
-- Check for null values
SELECT 
    COUNT(*) AS null_count
FROM customers
WHERE customer_id IS NULL OR name IS NULL;

-- Check for duplicates
SELECT 
    customer_id,
    COUNT(*) AS duplicate_count
FROM customers
GROUP BY customer_id
HAVING COUNT(*) > 1;
```

**Learning Outcome:** Implement data quality frameworks

---

### 21. Working with External Stages and S3
**Description:** Create and manage external stages for data integration.

```sql
CREATE EXTERNAL STAGE s3_stage
  URL = 's3://my-bucket/data/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
  FILE_FORMAT = (TYPE = 'PARQUET');

SELECT * FROM @s3_stage;
```

**Learning Outcome:** Integrate with cloud storage systems

---

### 22. Snowpipe for Continuous Data Ingestion
**Description:** Set up automated data loading.

```sql
CREATE PIPE customer_pipe AS
  COPY INTO customers
  FROM @s3_stage/customers/
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
  ON_ERROR = 'SKIP_FILE';

-- Pause/Resume pipe
ALTER PIPE customer_pipe SET PIPE_EXECUTION_PAUSED = TRUE;
```

**Learning Outcome:** Implement continuous data pipelines

---

### 23. Creating Materialized Views for Performance
**Description:** Pre-compute aggregations for faster queries.

```sql
CREATE MATERIALIZED VIEW customer_monthly_summary AS
SELECT 
    DATE_TRUNC('MONTH', order_date) AS month,
    customer_id,
    COUNT(*) AS order_count,
    SUM(order_amount) AS monthly_revenue
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date), customer_id;

-- Refresh materialized view
ALTER MATERIALIZED VIEW customer_monthly_summary REFRESH;
```

**Learning Outcome:** Optimize performance with materialized views

---

### 24. Implementing Role-Based Access Control (RBAC)
**Description:** Implement column-level and row-level security.

```sql
-- Column-level masking policy
CREATE MASKING POLICY email_masking AS (email_col VARCHAR)
RETURNS VARCHAR ->
    CASE
        WHEN CURRENT_ROLE() = 'ANALYST_ROLE' 
            THEN '*****@*****.com'
        ELSE email_col
    END;

ALTER TABLE customers
    MODIFY COLUMN email SET MASKING POLICY email_masking;
```

**Learning Outcome:** Implement security and compliance controls

---

### 25. Monitoring Queries and Resource Usage
**Description:** Track query performance and resource consumption.

```sql
-- View query history
SELECT 
    query_id,
    user_name,
    query_text,
    execution_time,
    bytes_scanned,
    credits_used
FROM snowflake.account_usage.query_history
WHERE execution_date >= DATEADD(DAY, -7, CURRENT_DATE())
ORDER BY execution_time DESC
LIMIT 100;
```

**Learning Outcome:** Optimize costs and performance through monitoring

---

## Advanced Level

### 26. Building Data Marts with Dimensional Modeling
**Description:** Create a complete star schema for business analytics.

```sql
-- Dimension table: Customer Dimension
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY,
    customer_id INT UNIQUE,
    name VARCHAR,
    email VARCHAR,
    country VARCHAR,
    city VARCHAR,
    valid_from DATE,
    valid_to DATE
);

-- Dimension table: Date Dimension
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    date_value DATE,
    year INT,
    month INT,
    day INT,
    quarter INT,
    week_num INT,
    is_weekend BOOLEAN
);

-- Fact table: Sales Facts
CREATE TABLE fact_sales (
    fact_id INT PRIMARY KEY,
    customer_key INT,
    date_key INT,
    product_key INT,
    sales_amount DECIMAL(10, 2),
    quantity INT
) CLUSTER BY (date_key, customer_key);
```

**Learning Outcome:** Build enterprise-scale data warehouses

---

### 27. Real-Time Analytics with Iceberg Tables
**Description:** Implement ACID transactions with Iceberg table format.

```sql
CREATE TABLE iceberg_transactions (
    transaction_id INT,
    customer_id INT,
    amount DECIMAL(10, 2),
    transaction_date TIMESTAMP
)
CLUSTER BY (transaction_date);

-- Insert with ACID guarantees
INSERT INTO iceberg_transactions VALUES
(1, 101, 250.00, CURRENT_TIMESTAMP()),
(2, 102, 500.00, CURRENT_TIMESTAMP());
```

**Learning Outcome:** Implement ACID-compliant data operations

---

### 28. Advanced Data Partitioning Strategies
**Description:** Implement partitioning for massive datasets.

```sql
CREATE TABLE transaction_data (
    transaction_id INT,
    customer_id INT,
    amount DECIMAL(10, 2),
    transaction_date DATE
)
CLUSTER BY (transaction_date, customer_id);

-- Query with partition pruning
SELECT COUNT(*) FROM transaction_data 
WHERE transaction_date >= '2026-01-01' 
  AND customer_id IN (SELECT customer_id FROM top_customers);
```

**Learning Outcome:** Optimize queries on massive datasets

---

### 29. Machine Learning Integration (PREDICT with Cortex)
**Description:** Integrate ML predictions using Snowflake Cortex.

```sql
-- Using Cortex for predictions
SELECT 
    customer_id,
    name,
    SNOWFLAKE.ML.PREDICT_CHURN(
        customer_id, 
        total_purchases, 
        last_purchase_date
    ) AS churn_probability
FROM customers
WHERE churn_probability > 0.7;
```

**Learning Outcome:** Integrate ML models into data workflows

---

### 30. Data Sharing Using Secure Data Exchange
**Description:** Share data securely across organizations.

```sql
-- Create a share for external partners
CREATE SHARE partner_share;

GRANT SELECT ON DATABASE analytics_db TO SHARE partner_share;
GRANT SELECT ON SCHEMA analytics_db.public TO SHARE partner_share;
GRANT SELECT ON TABLE analytics_db.public.customer_summary TO SHARE partner_share;

-- Add consumer account
ALTER SHARE partner_share ADD ACCOUNTS = 'partner_account';
```

**Learning Outcome:** Implement secure data sharing

---

### 31. Building ETL Pipelines with Tasks and Streams
**Description:** Implement serverless ETL with Snowflake Tasks.

```sql
-- Create a stream to track changes
CREATE STREAM customer_changes ON TABLE customers;

-- Create a task that runs on schedule
CREATE TASK process_customer_changes
  WAREHOUSE = analytics_wh
  SCHEDULE = '1 HOUR'
AS
  INSERT INTO customer_audit
  SELECT 
      CURRENT_TIMESTAMP() as process_time,
      metadata$action,
      metadata$isupdate,
      *
  FROM customer_changes;

ALTER TASK process_customer_changes RESUME;
```

**Learning Outcome:** Build serverless data pipelines

---

### 32. Dynamic Data Masking with Policies
**Description:** Implement column-level security policies.

```sql
CREATE MASKING POLICY salary_masking AS (salary DECIMAL(10, 2))
RETURNS DECIMAL(10, 2) ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'HR_MANAGER') THEN salary
        ELSE NULL
    END;

ALTER TABLE employees MODIFY COLUMN salary SET MASKING POLICY salary_masking;
```

**Learning Outcome:** Implement fine-grained security controls

---

### 33. Advanced JSON Processing and Normalization
**Description:** Parse complex nested JSON structures.

```sql
CREATE TABLE nested_events (
    event_id INT,
    event_data VARIANT
);

SELECT 
    event_data:user:id::INT AS user_id,
    event_data:user:name::VARCHAR AS user_name,
    event_data:properties AS properties,
    VALUE AS property_value
FROM nested_events,
LATERAL FLATTEN(input => event_data:properties)
WHERE event_data:event_type = 'purchase';
```

**Learning Outcome:** Master complex data transformations

---

### 34. Implementing Feature Stores
**Description:** Create feature stores for ML operations.

```sql
CREATE TABLE customer_features (
    customer_id INT,
    feature_date DATE,
    total_purchases INT,
    avg_order_value DECIMAL(10, 2),
    days_since_last_purchase INT,
    customer_lifetime_value DECIMAL(10, 2),
    churn_risk_score DECIMAL(5, 4)
)
CLUSTER BY (feature_date, customer_id);

-- Create view for ML model access
CREATE VIEW feature_store_current AS
SELECT * FROM customer_features
WHERE feature_date = CURRENT_DATE();
```

**Learning Outcome:** Build infrastructure for ML operations

---

### 35. Dynamic Data Warehouse Scaling
**Description:** Scale warehouses based on workload demands.

```sql
-- Create auto-scaling warehouse
CREATE WAREHOUSE compute_auto_scale
  WAREHOUSE_SIZE = 'MEDIUM'
  MAX_CLUSTER_COUNT = 10
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD';

-- Monitor scaling
SHOW WAREHOUSES LIKE 'compute_auto_scale';
```

**Learning Outcome:** Optimize cost with dynamic scaling

---

### 36. Cross-Database Joins and References
**Description:** Query across multiple databases.

```sql
SELECT 
    p.product_id,
    p.name,
    s.total_sales
FROM primary_db.public.products p
FULL OUTER JOIN analytics_db.public.sales_summary s
    ON p.product_id = s.product_id;
```

**Learning Outcome:** Federate data across databases

---

### 37. Implementing Complex Business Logic with CTEs
**Description:** Build complex queries using Common Table Expressions.

```sql
WITH monthly_sales AS (
    SELECT 
        DATE_TRUNC('MONTH', order_date) AS month,
        customer_id,
        SUM(order_amount) AS monthly_total
    FROM orders
    GROUP BY DATE_TRUNC('MONTH', order_date), customer_id
),
customer_avg AS (
    SELECT 
        customer_id,
        AVG(monthly_total) AS avg_monthly_sales
    FROM monthly_sales
    GROUP BY customer_id
)
SELECT 
    ms.customer_id,
    ms.month,
    ms.monthly_total,
    ca.avg_monthly_sales,
    ROUND((ms.monthly_total / ca.avg_monthly_sales - 1) * 100, 2) AS percent_above_avg
FROM monthly_sales ms
JOIN customer_avg ca ON ms.customer_id = ca.customer_id
WHERE ms.monthly_total > ca.avg_monthly_sales;
```

**Learning Outcome:** Master complex analytical queries

---

### 38. Implementing Multi-Tenant Data Architecture
**Description:** Design for SaaS multi-tenancy.

```sql
-- Account-level isolation
CREATE TABLE tenant_data (
    tenant_id VARCHAR,
    data_id INT,
    content VARIANT,
    created_at TIMESTAMP
)
CLUSTER BY (tenant_id, created_at);

-- Row-level security for tenants
CREATE ROW ACCESS POLICY tenant_isolation AS (tenant_id VARCHAR)
RETURNS BOOLEAN ->
    TENANT_ID = CURRENT_USER()
    OR CURRENT_ROLE() = 'ADMIN';

ALTER TABLE tenant_data ADD ROW ACCESS POLICY tenant_isolation ON (tenant_id);
```

**Learning Outcome:** Build multi-tenant platforms

---

### 39. Implementing Budget Alerts and Cost Management
**Description:** Monitor and control Snowflake costs.

```sql
-- Create table for cost tracking
CREATE TABLE cost_alerts (
    alert_id INT,
    date_hour TIMESTAMP,
    warehouse_name VARCHAR,
    credits_used DECIMAL(10, 2),
    estimated_cost DECIMAL(10, 2)
);

-- Insert hourly cost data
INSERT INTO cost_alerts
SELECT 
    ROW_NUMBER() OVER (ORDER BY start_time),
    start_time,
    warehouse_name,
    credits_used,
    credits_used * 4.00 AS estimated_cost
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(HOUR, -24, CURRENT_TIMESTAMP());
```

**Learning Outcome:** Implement cost governance

---

### 40. Building Data Governance Frameworks
**Description:** Implement data lineage and governance.

```sql
-- Create data lineage table
CREATE TABLE data_lineage (
    lineage_id INT,
    source_object VARCHAR,
    target_object VARCHAR,
    transformation_type VARCHAR,
    last_modified TIMESTAMP,
    owner VARCHAR
);

-- Create data catalog
CREATE TABLE data_catalog (
    object_id INT,
    object_name VARCHAR,
    object_type VARCHAR,
    description VARCHAR,
    owner VARCHAR,
    classification VARCHAR,
    created_date TIMESTAMP
);
```

**Learning Outcome:** Implement enterprise governance

---

## Expert Level

### 41. Building Real-Time OLAP Cubes
**Description:** Create multi-dimensional analysis structures.

```sql
CREATE TABLE sales_cube (
    cube_id INT,
    customer_dimension_key INT,
    product_dimension_key INT,
    time_dimension_key INT,
    geography_dimension_key INT,
    sales_amount DECIMAL(15, 2),
    quantity INT,
    margin DECIMAL(15, 2),
    created_timestamp TIMESTAMP
)
CLUSTER BY (time_dimension_key, customer_dimension_key, product_dimension_key);

-- Aggregation query for OLAP
SELECT 
    cd.country,
    pd.category,
    TIME_DIMENSION:year,
    SUM(sales_cube.sales_amount) AS total_sales,
    AVG(sales_cube.margin) AS avg_margin,
    COUNT(*) AS transaction_count
FROM sales_cube
JOIN dim_customer cd ON sales_cube.customer_dimension_key = cd.customer_key
JOIN dim_product pd ON sales_cube.product_dimension_key = pd.product_key
GROUP BY CUBE(cd.country, pd.category);
```

**Learning Outcome:** Build enterprise-scale OLAP systems

---

### 42. Advanced Query Optimization with Query Rewrite
**Description:** Optimize complex queries automatically.

```sql
-- Original query
EXPLAIN
SELECT 
    c.customer_id,
    COUNT(o.order_id) AS order_count,
    SUM(o.order_amount) AS total_amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id
HAVING COUNT(o.order_id) > 10;

-- Snowflake automatically optimizes join order and predicate pushdown
```

**Learning Outcome:** Master query execution optimization

---

### 43. Implementing Data Vault 2.0 Architecture
**Description:** Build scalable data vault warehouses.

```sql
-- Hub table
CREATE TABLE hub_customer (
    customer_hk BINARY,
    customer_id VARCHAR,
    load_date TIMESTAMP,
    record_source VARCHAR
);

-- Satellite table
CREATE TABLE sat_customer_details (
    customer_hk BINARY,
    load_date TIMESTAMP,
    load_end_date TIMESTAMP,
    is_current BOOLEAN,
    name VARCHAR,
    email VARCHAR,
    address VARCHAR,
    record_source VARCHAR
);

-- Link table
CREATE TABLE link_customer_product (
    customer_product_hk BINARY,
    customer_hk BINARY,
    product_hk BINARY,
    load_date TIMESTAMP,
    record_source VARCHAR
);
```

**Learning Outcome:** Build scalable data vault systems

---

### 44. Implementing Advanced Stream Processing
**Description:** Build real-time streaming analytics.

```sql
-- Create stream from external source
CREATE STREAM external_events_stream
  COPY GRANTS
  ON TABLE raw_events;

-- Create dynamic task for stream processing
CREATE TASK process_events_realtime
  WAREHOUSE = realtime_wh
  SCHEDULE = '5 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('external_events_stream')
AS
  INSERT INTO processed_events
  SELECT 
      event_id,
      CURRENT_TIMESTAMP() AS processed_at,
      es.* EXCLUDE (metadata$action, metadata$isupdate, metadata$row_id)
  FROM external_events_stream es;
```

**Learning Outcome:** Implement streaming data pipelines

---

### 45. Building Custom ML Models with Python UDFs
**Description:** Implement Python-based ML in Snowflake.

```sql
CREATE FUNCTION predict_customer_churn(
    customer_data OBJECT
)
RETURNS OBJECT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('scikit-learn', 'pandas')
HANDLER = 'predict_churn'
AS
$$
import pickle
import pandas as pd
from sklearn.preprocessing import StandardScaler

def predict_churn(customer_data):
    # Load pre-trained model
    model = pickle.loads(get_model_from_stage())
    
    # Prepare features
    features = pd.DataFrame([customer_data])
    scaler = StandardScaler()
    scaled_features = scaler.fit_transform(features)
    
    # Make prediction
    prediction = model.predict(scaled_features)[0]
    probability = model.predict_proba(scaled_features)[0][1]
    
    return {
        'prediction': int(prediction),
        'probability': float(probability)
    }
$$;
```

**Learning Outcome:** Integrate advanced ML models

---

### 46. Implementing Advanced Encryption and Key Management
**Description:** Implement end-to-end encryption.

```sql
-- Create encrypted column with customer-managed keys
CREATE OR REPLACE TABLE sensitive_data (
    record_id INT,
    encrypted_ssn VARCHAR ENCRYPTED WITH (
        TYPE = 'DETERMINISTIC',
        ALGORITHM = 'AES-256-CBC',
        KEY_IDENTIFIER = 'customer_key_id_1'
    ),
    encrypted_credit_card VARCHAR ENCRYPTED WITH (
        TYPE = 'RANDOM',
        ALGORITHM = 'AES-256-GCM'
    )
);

-- Query encrypted columns
SELECT 
    record_id,
    DECRYPT(encrypted_ssn, 'customer_key_id_1') AS ssn
FROM sensitive_data
WHERE CURRENT_ROLE() = 'SECURITY_OFFICER';
```

**Learning Outcome:** Implement enterprise security controls

---

### 47. Implementing Change Data Capture (CDC) Pipeline
**Description:** Build comprehensive CDC pipelines.

```sql
-- Create source table and stream
CREATE TABLE source_customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR,
    status VARCHAR,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

CREATE STREAM source_customers_cdc ON TABLE source_customers;

-- Create CDC processor
CREATE TASK cdc_processor
  WAREHOUSE = etl_wh
  SCHEDULE = '1 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('source_customers_cdc')
AS
  BEGIN
    INSERT INTO customer_cdc_log
    SELECT 
        CURRENT_TIMESTAMP() AS processed_at,
        METADATA$ACTION AS action_type,
        METADATA$ISUPDATE AS is_update,
        *
    FROM source_customers_cdc;
  END;
```

**Learning Outcome:** Implement CDC at scale

---

### 48. Advanced Performance Tuning with Statistics and Hints
**Description:** Optimize performance with detailed analysis.

```sql
-- Analyze table statistics
ANALYZE TABLE large_transactions
COMPUTE STATS;

-- Use optimizer hints for specific query plans
SELECT /*+ USE_HASH_JOIN(o, c) */
    c.customer_id,
    COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.country = 'USA'
GROUP BY c.customer_id;

-- Gather query profile for optimization
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

**Learning Outcome:** Master advanced optimization techniques

---

### 49. Building Custom Metadata and Audit Systems
**Description:** Implement comprehensive audit trails.

```sql
-- Create audit table
CREATE TABLE universal_audit_log (
    audit_id INT AUTOINCREMENT,
    event_timestamp TIMESTAMP,
    user_name VARCHAR,
    database_name VARCHAR,
    table_name VARCHAR,
    operation_type VARCHAR,
    old_values VARIANT,
    new_values VARIANT,
    change_reason VARCHAR
);

-- Create audit trigger function
CREATE OR REPLACE PROCEDURE audit_changes()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    INSERT INTO universal_audit_log
    SELECT 
        SEQ4(),
        CURRENT_TIMESTAMP(),
        CURRENT_USER(),
        DATABASE_NAME,
        TABLE_NAME,
        OPERATION,
        BEFORE_VALUES,
        AFTER_VALUES,
        CHANGE_REASON
    FROM system_audit_trail
    WHERE event_timestamp > DATEADD(HOUR, -1, CURRENT_TIMESTAMP());
    
    RETURN 'Audit log updated';
END;
$$;
```

**Learning Outcome:** Implement comprehensive audit systems

---

### 50. Building End-to-End Data Platform Architecture
**Description:** Design and implement a complete data platform.

```sql
-- Ingestion layer
CREATE SCHEMA ingestion_layer;

CREATE TABLE ingestion_layer.raw_data (
    raw_id INT AUTOINCREMENT,
    source_system VARCHAR,
    raw_payload VARIANT,
    ingestion_timestamp TIMESTAMP,
    data_hash VARCHAR,
    is_processed BOOLEAN DEFAULT FALSE
) CLUSTER BY (source_system, ingestion_timestamp);

-- Processing layer
CREATE SCHEMA processing_layer;

CREATE TABLE processing_layer.staging_data (
    staging_id INT AUTOINCREMENT,
    source_raw_id INT,
    cleaned_data VARIANT,
    data_quality_score DECIMAL(5, 4),
    processing_timestamp TIMESTAMP
) CLUSTER BY (processing_timestamp);

-- Analytics layer
CREATE SCHEMA analytics_layer;

CREATE TABLE analytics_layer.fact_consolidated (
    fact_id INT,
    customer_key INT,
    product_key INT,
    date_key INT,
    metrics VARIANT,
    created_timestamp TIMESTAMP
) CLUSTER BY (date_key, customer_key);

-- Create orchestration pipeline
CREATE TASK platform_ingestion_task
  WAREHOUSE = platform_wh
  SCHEDULE = '5 MINUTE'
AS
  INSERT INTO processing_layer.staging_data
  SELECT 
      SEQ4(),
      raw_id,
      raw_payload,
      1.0,
      CURRENT_TIMESTAMP()
  FROM ingestion_layer.raw_data
  WHERE is_processed = FALSE;

CREATE TASK platform_analytics_task
  WAREHOUSE = platform_wh
  SCHEDULE = '1 HOUR'
  AFTER platform_ingestion_task
AS
  INSERT INTO analytics_layer.fact_consolidated
  SELECT 
      ROW_NUMBER() OVER (ORDER BY s.staging_id),
      s.staging_data:customer_id::INT,
      s.staging_data:product_id::INT,
      DATE(s.processing_timestamp)::INT,
      s.staging_data,
      CURRENT_TIMESTAMP()
  FROM processing_layer.staging_data s;
```

**Learning Outcome:** Design enterprise-scale data platforms

---

## Summary

This guide covers 50 progressively complex Snowflake use cases:

- **Beginner (1-10):** Foundation building with databases, tables, queries, and basic operations
- **Intermediate (11-25):** Advanced queries, performance optimization, and data management
- **Advanced (26-40):** Enterprise architectures, governance, and optimization
- **Expert (41-50):** Cutting-edge implementations for large-scale systems

### Key Takeaways

1. **Start Simple:** Master basic operations before moving to advanced features
2. **Performance First:** Always consider clustering, partitioning, and materialized views
3. **Security Always:** Implement row/column-level security and encryption from the start
4. **Monitor Costs:** Track credit usage and optimize warehouse sizing
5. **Automate Everything:** Use Tasks and Streams for serverless pipelines
6. **Plan for Scale:** Build with multi-tenancy and data governance in mind

### Resources

- [Snowflake Official Documentation](https://docs.snowflake.com)
- [Snowflake Community](https://community.snowflake.com)
- [Snowflake University](https://university.snowflake.com)
- [Advanced SQL Patterns](https://docs.snowflake.com/en/sql-reference/)

---

**Last Updated:** February 18, 2026
