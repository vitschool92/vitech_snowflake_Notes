# Use Case 27-40: Advanced Snowflake Use Cases

## Use Case 27: Real-Time Analytics with Iceberg Tables

```sql
-- Create Iceberg table with ACID transactions
CREATE TABLE iceberg_transactions (
    transaction_id INT,
    customer_id INT,
    amount DECIMAL(10,2),
    transaction_date TIMESTAMP
)
CLUSTER BY (transaction_date);

-- Insert with ACID guarantees
INSERT INTO iceberg_transactions VALUES (1, 101, 250.00, CURRENT_TIMESTAMP());

-- Time travel on Iceberg tables
SELECT * FROM iceberg_transactions AT (OFFSET => -3600);
```

## Use Case 28: Advanced Data Partitioning Strategies

```sql
-- Create highly partitioned table
CREATE TABLE transaction_data (
    transaction_id INT,
    customer_id INT,
    amount DECIMAL(10,2),
    transaction_date DATE
)
CLUSTER BY (transaction_date, customer_id);

-- Query with partition pruning
SELECT COUNT(*) FROM transaction_data 
WHERE transaction_date >= '2026-01-01' 
  AND customer_id > 0;
```

## Use Case 29: Machine Learning Integration (PREDICT with Cortex)

```sql
-- Use Snowflake Cortex for predictions
SELECT 
    customer_id,
    SNOWFLAKE.ML.PREDICT_CHURN(customer_id, total_purchases) AS churn_probability
FROM customers
WHERE total_purchases > 10;
```

## Use Case 30: Data Sharing Using Secure Data Exchange

```sql
-- Create share
CREATE SHARE partner_share;

-- Grant access
GRANT SELECT ON DATABASE analytics_db TO SHARE partner_share;
GRANT SELECT ON SCHEMA analytics_db.public TO SHARE partner_share;

-- Add consumer account
ALTER SHARE partner_share ADD ACCOUNTS = 'partner_account';
```

## Use Case 31: Building ETL Pipelines with Tasks and Streams

```sql
-- Create stream for change data capture
CREATE STREAM customer_changes ON TABLE customers;

-- Create task for ETL
CREATE TASK process_changes
  WAREHOUSE = etl_wh
  SCHEDULE = 'USING CRON 0 * * * * UTC'
WHEN SYSTEM$STREAM_HAS_DATA('customer_changes')
AS
  INSERT INTO customer_audit
  SELECT * EXCLUDE (metadata$action, metadata$isupdate, metadata$row_id)
  FROM customer_changes;

ALTER TASK process_changes RESUME;
```

## Use Case 32: Dynamic Data Masking with Policies

```sql
-- Create comprehensive masking policy
CREATE MASKING POLICY pii_mask AS (pii_value VARCHAR)
RETURNS VARCHAR ->
    CASE
        WHEN CURRENT_ROLE() = 'ADMIN' THEN pii_value
        WHEN CURRENT_ROLE() LIKE '%ANALYST%' THEN SUBSTRING(pii_value, 1, 3) || '****'
        ELSE '****'
    END;

-- Apply to columns
ALTER TABLE customers MODIFY COLUMN ssn SET MASKING POLICY pii_mask;
ALTER TABLE customers MODIFY COLUMN credit_card SET MASKING POLICY pii_mask;
```

## Use Case 33: Advanced JSON Processing and Normalization

```sql
-- Flatten complex nested JSON
SELECT 
    event_id,
    f1.key AS top_level_key,
    f2.key AS nested_key,
    f2.value AS value
FROM events,
LATERAL FLATTEN(input => event_data) f1,
LATERAL FLATTEN(input => f1.value) f2
WHERE f1.key NOT LIKE 'metadata%';
```

## Use Case 34: Implementing Feature Stores

```sql
-- Create feature store
CREATE TABLE customer_features (
    customer_id INT,
    feature_date DATE,
    feature_1 DECIMAL(10,2),
    feature_2 INT,
    feature_3 VARCHAR(50),
    created_timestamp TIMESTAMP
)
CLUSTER BY (feature_date, customer_id);

-- Create view for ML
CREATE VIEW feature_store_current AS
SELECT * FROM customer_features
WHERE feature_date = CURRENT_DATE();
```

## Use Case 35: Dynamic Data Warehouse Scaling

```sql
-- Create auto-scaling warehouse
CREATE WAREHOUSE auto_scale_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  MAX_CLUSTER_COUNT = 10
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE;
```

## Use Case 36: Cross-Database Joins and References

```sql
-- Query across databases
SELECT 
    p.product_id,
    p.name,
    s.total_sales
FROM primary_db.public.products p
FULL OUTER JOIN analytics_db.public.sales_summary s
    ON p.product_id = s.product_id;
```

## Use Case 37: Implementing Complex Business Logic with CTEs

```sql
-- Complex CTE logic
WITH monthly_metrics AS (
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
        AVG(monthly_total) AS avg_monthly
    FROM monthly_metrics
    GROUP BY customer_id
)
SELECT 
    m.customer_id,
    m.month,
    m.monthly_total,
    c.avg_monthly,
    ROUND((m.monthly_total / c.avg_monthly - 1) * 100, 2) AS percent_above_avg
FROM monthly_metrics m
JOIN customer_avg c ON m.customer_id = c.customer_id
WHERE m.monthly_total > c.avg_monthly;
```

## Use Case 38: Implementing Multi-Tenant Data Architecture

```sql
-- Create tenant-isolated table
CREATE TABLE tenant_data (
    tenant_id VARCHAR,
    data_id INT,
    content VARIANT,
    created_at TIMESTAMP
)
CLUSTER BY (tenant_id, created_at);

-- Row access policy for tenants
CREATE ROW ACCESS POLICY tenant_filter AS (tenant_id VARCHAR)
RETURNS BOOLEAN ->
    CURRENT_USER() = tenant_id OR CURRENT_ROLE() = 'ADMIN';

ALTER TABLE tenant_data ADD ROW ACCESS POLICY tenant_filter ON (tenant_id);
```

## Use Case 39: Implementing Budget Alerts and Cost Management

```sql
-- Create cost tracking
CREATE TABLE cost_tracking (
    warehouse_name VARCHAR,
    hour_date TIMESTAMP,
    credits_used DECIMAL(10,2),
    estimated_cost DECIMAL(10,2)
);

-- Monitor costs
INSERT INTO cost_tracking
SELECT 
    warehouse_name,
    start_time,
    credits_used,
    credits_used * 4.0 AS estimated_cost
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD(HOUR, -1, CURRENT_TIMESTAMP());

-- Alert on high costs
CREATE TASK cost_alert_task
  WAREHOUSE = analytics_wh
  SCHEDULE = 'USING CRON 0 * * * * UTC'
AS
  INSERT INTO cost_alerts
  SELECT * FROM cost_tracking
  WHERE estimated_cost > 100;
```

## Use Case 40: Building Data Governance Frameworks

```sql
-- Create data lineage table
CREATE TABLE data_lineage (
    lineage_id INT AUTOINCREMENT,
    source_table VARCHAR,
    target_table VARCHAR,
    transformation_type VARCHAR,
    owner VARCHAR,
    last_modified TIMESTAMP
);

-- Create data catalog
CREATE TABLE data_catalog (
    object_id INT AUTOINCREMENT,
    object_name VARCHAR,
    object_type VARCHAR,
    description VARCHAR,
    owner VARCHAR,
    classification VARCHAR(20),
    created_date TIMESTAMP
);

-- Insert catalog entries
INSERT INTO data_catalog VALUES 
(1, 'customers', 'TABLE', 'Master customer data', 'data_team', 'INTERNAL', CURRENT_TIMESTAMP()),
(2, 'orders', 'TABLE', 'Order transactions', 'data_team', 'INTERNAL', CURRENT_TIMESTAMP());
```

---

**Last Updated:** February 18, 2026
