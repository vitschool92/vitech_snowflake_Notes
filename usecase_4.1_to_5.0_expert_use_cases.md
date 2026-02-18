# Use Case 41-50: Expert Snowflake Use Cases

## Use Case 41: Building Real-Time OLAP Cubes

```sql
-- Create multi-dimensional cube
CREATE TABLE sales_cube (
    cube_id INT,
    customer_key INT,
    product_key INT,
    time_key INT,
    region_key INT,
    sales_amount DECIMAL(15,2),
    quantity INT,
    margin DECIMAL(15,2),
    created_timestamp TIMESTAMP
)
CLUSTER BY (time_key, customer_key, product_key);

-- OLAP query with GROUP BY CUBE
SELECT 
    c.country,
    p.category,
    DATE_TRUNC('MONTH', d.date_value),
    SUM(s.sales_amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM sales_cube s
JOIN dim_customer c ON s.customer_key = c.customer_key
JOIN dim_product p ON s.product_key = p.product_key
JOIN dim_date d ON s.time_key = d.date_key
GROUP BY CUBE(c.country, p.category, DATE_TRUNC('MONTH', d.date_value))
ORDER BY total_sales DESC;
```

## Use Case 42: Advanced Query Optimization with Query Rewrite

```sql
-- Use EXPLAIN to see optimization
EXPLAIN
SELECT 
    c.customer_id,
    COUNT(o.order_id) AS order_count,
    SUM(o.order_amount) AS total
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id
HAVING COUNT(o.order_id) > 5;

-- Query profile for detailed metrics
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

## Use Case 43: Implementing Data Vault 2.0 Architecture

```sql
-- Hub table (business keys)
CREATE TABLE hub_customer (
    customer_hk BINARY,
    customer_id VARCHAR,
    load_date TIMESTAMP,
    record_source VARCHAR
);

-- Satellite table (attributes with history)
CREATE TABLE sat_customer_details (
    customer_hk BINARY,
    load_date TIMESTAMP,
    load_end_date TIMESTAMP,
    is_current BOOLEAN,
    name VARCHAR,
    email VARCHAR,
    record_source VARCHAR
);

-- Link table (relationships)
CREATE TABLE link_customer_product (
    link_hk BINARY,
    customer_hk BINARY,
    product_hk BINARY,
    load_date TIMESTAMP,
    record_source VARCHAR
);
```

## Use Case 44: Implementing Advanced Stream Processing

```sql
-- Create stream with multiple change types
CREATE STREAM events_stream ON TABLE raw_events;

-- Process with conditional logic
CREATE TASK stream_processor
  WAREHOUSE = realtime_wh
  SCHEDULE = '1 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('events_stream')
AS
  INSERT INTO processed_events
  SELECT 
      event_id,
      metadata$action AS action_type,
      metadata$isupdate AS is_update,
      CURRENT_TIMESTAMP() AS processed_at
  FROM events_stream;

ALTER TASK stream_processor RESUME;
```

## Use Case 45: Building Custom ML Models with Python UDFs

```sql
-- Create Python UDF for ML predictions
CREATE FUNCTION predict_customer_churn(
    customer_data OBJECT
)
RETURNS OBJECT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('scikit-learn', 'pandas', 'numpy')
HANDLER = 'predict'
AS
$$
import pickle
import pandas as pd
from sklearn.preprocessing import StandardScaler

def predict(data):
    # Load model
    model = load_model_from_stage()
    
    # Prepare features
    features = pd.DataFrame([data])
    scaled = StandardScaler().fit_transform(features)
    
    # Predict
    prediction = model.predict(scaled)[0]
    probability = model.predict_proba(scaled)[0][1]
    
    return {
        'prediction': int(prediction),
        'probability': float(probability),
        'risk_level': 'HIGH' if probability > 0.7 else 'MEDIUM' if probability > 0.4 else 'LOW'
    }
$$;
```

## Use Case 46: Implementing Advanced Encryption and Key Management

```sql
-- Create encrypted column with customer-managed keys
CREATE TABLE sensitive_data (
    record_id INT,
    encrypted_ssn VARCHAR,
    encrypted_credit_card VARCHAR,
    encryption_key_id VARCHAR
);

-- Insert encrypted data
INSERT INTO sensitive_data
SELECT 
    record_id,
    ENCRYPT('AES-256-CBC', ssn_value, 'encryption_key_1') AS encrypted_ssn,
    ENCRYPT('AES-256-GCM', credit_card_value, 'encryption_key_2') AS encrypted_credit_card,
    'encryption_key_1' AS encryption_key_id
FROM raw_data;

-- Decrypt for authorized users
SELECT 
    record_id,
    DECRYPT('AES-256-CBC', encrypted_ssn, 'encryption_key_1') AS ssn
FROM sensitive_data
WHERE CURRENT_ROLE() = 'SECURITY_OFFICER';
```

## Use Case 47: Implementing Change Data Capture (CDC) Pipeline

```sql
-- Create CDC with stream
CREATE TABLE source_system (
    id INT PRIMARY KEY,
    name VARCHAR,
    updated_at TIMESTAMP
);

CREATE STREAM source_stream ON TABLE source_system;

-- CDC processor with history
CREATE TASK cdc_processor
  WAREHOUSE = etl_wh
  SCHEDULE = '5 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('source_stream')
AS
  INSERT INTO change_log
  SELECT 
      CURRENT_TIMESTAMP() AS captured_at,
      metadata$action AS action_type,
      metadata$isupdate AS is_update,
      *
  FROM source_stream;

ALTER TASK cdc_processor RESUME;
```

## Use Case 48: Advanced Performance Tuning with Statistics and Hints

```sql
-- Analyze table statistics
ANALYZE TABLE large_transactions COMPUTE STATS;

-- Use optimizer hints
SELECT /*+ USE_HASH_JOIN(o, c) */
    c.customer_id,
    COUNT(o.order_id) AS order_count
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;

-- Get query profile with statistics
EXPLAIN ANALYZE
SELECT * FROM sales_data
WHERE region = 'USA'
  AND sale_date >= '2025-01-01';
```

## Use Case 49: Building Custom Metadata and Audit Systems

```sql
-- Universal audit log
CREATE TABLE audit_log (
    audit_id INT AUTOINCREMENT,
    event_timestamp TIMESTAMP,
    user_name VARCHAR,
    operation_type VARCHAR,
    object_type VARCHAR,
    object_name VARCHAR,
    old_values VARIANT,
    new_values VARIANT,
    ip_address VARCHAR
);

-- Audit trigger procedure
CREATE OR REPLACE PROCEDURE log_changes()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    INSERT INTO audit_log
    SELECT 
        SEQ4(),
        CURRENT_TIMESTAMP(),
        CURRENT_USER(),
        OPERATION,
        OBJECT_TYPE,
        OBJECT_NAME,
        OLD_VALUES,
        NEW_VALUES,
        NULL
    FROM system_audit_trail
    WHERE event_timestamp > DATEADD(HOUR, -1, CURRENT_TIMESTAMP());
    
    RETURN 'Audit log updated: ' || ROW_COUNT() || ' records';
END;
$$;
```

## Use Case 50: Building End-to-End Data Platform Architecture

```sql
-- Complete 3-tier architecture

-- ============== INGESTION LAYER ==============
CREATE SCHEMA ingestion_layer;

CREATE TABLE ingestion_layer.raw_data (
    raw_id INT AUTOINCREMENT,
    source_system VARCHAR,
    raw_payload VARIANT,
    ingestion_timestamp TIMESTAMP,
    is_processed BOOLEAN DEFAULT FALSE
)
CLUSTER BY (source_system, ingestion_timestamp);

CREATE PIPE ingestion_layer.data_pipe AS
  COPY INTO ingestion_layer.raw_data (source_system, raw_payload, ingestion_timestamp)
  FROM @external_stage/
  FILE_FORMAT = json_format
  ON_ERROR = 'SKIP_FILE';

-- ============== PROCESSING LAYER ==============
CREATE SCHEMA processing_layer;

CREATE TABLE processing_layer.staging_data (
    staging_id INT AUTOINCREMENT,
    source_raw_id INT,
    cleaned_data VARIANT,
    quality_score DECIMAL(5,4),
    processing_timestamp TIMESTAMP
)
CLUSTER BY (processing_timestamp);

CREATE STREAM processing_layer.staging_stream 
  ON TABLE ingestion_layer.raw_data;

CREATE TASK processing_layer.process_raw_data
  WAREHOUSE = etl_wh
  SCHEDULE = '5 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('processing_layer.staging_stream')
AS
  INSERT INTO processing_layer.staging_data
  SELECT 
      SEQ4(),
      raw_id,
      raw_payload,
      1.0,
      CURRENT_TIMESTAMP()
  FROM ingestion_layer.raw_data
  WHERE is_processed = FALSE
  LIMIT 1000;

-- ============== ANALYTICS LAYER ==============
CREATE SCHEMA analytics_layer;

CREATE TABLE analytics_layer.fact_consolidated (
    fact_id INT,
    customer_key INT,
    product_key INT,
    date_key INT,
    metrics VARIANT,
    created_timestamp TIMESTAMP
)
CLUSTER BY (date_key, customer_key);

CREATE MATERIALIZED VIEW analytics_layer.sales_summary AS
SELECT 
    DATE_TRUNC('DAY', created_timestamp) AS sales_day,
    SUM(metrics:amount::DECIMAL) AS daily_sales,
    COUNT(*) AS transaction_count
FROM analytics_layer.fact_consolidated
GROUP BY DATE_TRUNC('DAY', created_timestamp);

-- ============== ORCHESTRATION ==============
CREATE TASK analytics_layer.refresh_summary
  WAREHOUSE = analytics_wh
  SCHEDULE = 'USING CRON 0 2 * * * UTC'
  AFTER processing_layer.process_raw_data
AS
  ALTER MATERIALIZED VIEW analytics_layer.sales_summary REFRESH;

-- ============== GOVERNANCE ==============
CREATE TABLE governance.data_lineage (
    lineage_id INT AUTOINCREMENT,
    source_table VARCHAR,
    target_table VARCHAR,
    transformation_logic VARCHAR,
    owner VARCHAR,
    last_modified TIMESTAMP,
    PRIMARY KEY (lineage_id)
);

CREATE TABLE governance.quality_metrics (
    metric_id INT AUTOINCREMENT,
    table_name VARCHAR,
    metric_name VARCHAR,
    metric_value DECIMAL(10,4),
    measured_timestamp TIMESTAMP
);

-- ============== SECURITY ==============
CREATE ROLE analyst_role;
GRANT SELECT ON SCHEMA analytics_layer.* TO ROLE analyst_role;

CREATE ROLE etl_role;
GRANT SELECT, INSERT, UPDATE ON SCHEMA ingestion_layer.* TO ROLE etl_role;
GRANT SELECT, INSERT, UPDATE ON SCHEMA processing_layer.* TO ROLE etl_role;

-- ============== RESUME ALL TASKS ==============
ALTER TASK processing_layer.process_raw_data RESUME;
ALTER TASK analytics_layer.refresh_summary RESUME;

-- Complete data platform ready for production
SHOW TASKS;
```

---

**Last Updated:** February 18, 2026

## Summary

This expert use case demonstrates building a complete production-grade data platform with:
- **Ingestion:** Automated Snowpipe loading
- **Processing:** Stream-based transformations
- **Analytics:** Optimized dimensional models
- **Governance:** Audit trails and quality metrics
- **Security:** Role-based access control
- **Orchestration:** Task dependencies and scheduling

This architecture can handle:
- Millions of records per day
- Real-time processing
- Complex transformations
- Enterprise governance
- Compliance requirements
- Cost optimization
