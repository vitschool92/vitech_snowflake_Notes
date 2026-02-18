# Use Case 20: Data Quality Monitoring with Automated Queries

## Problem Description

Data quality is critical. Automated checks should:

1. Identify null values
2. Detect duplicates
3. Validate formats
4. Check value ranges
5. Monitor freshness

## Business Context

A company needs to ensure data quality by:
- Alerting on null values in key fields
- Detecting duplicate records
- Validating email formats
- Monitoring data freshness
- Tracking quality metrics

## Solution

### NULL Value Checks

```sql
-- Find null values in required fields
SELECT 
    COUNT(*) AS null_count,
    COUNT(*) * 100.0 / (SELECT COUNT(*) FROM customers) AS null_percentage
FROM customers
WHERE customer_id IS NULL OR name IS NULL OR email IS NULL;

-- Detailed null analysis
SELECT 
    SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_customer_id,
    SUM(CASE WHEN name IS NULL THEN 1 ELSE 0 END) AS null_name,
    SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) AS null_email,
    SUM(CASE WHEN created_date IS NULL THEN 1 ELSE 0 END) AS null_created_date
FROM customers;
```

### Duplicate Detection

```sql
-- Find duplicate customer IDs
SELECT 
    customer_id,
    COUNT(*) AS occurrence_count
FROM customers
GROUP BY customer_id
HAVING COUNT(*) > 1
ORDER BY occurrence_count DESC;

-- Find duplicate emails
SELECT 
    email,
    COUNT(*) AS occurrence_count,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM customers
WHERE email IS NOT NULL
GROUP BY email
HAVING COUNT(*) > 1;

-- Find records with identical values
WITH duplicates AS (
    SELECT 
        customer_id,
        name,
        email,
        ROW_NUMBER() OVER (PARTITION BY name, email ORDER BY customer_id) AS rn
    FROM customers
)
SELECT * FROM duplicates WHERE rn > 1;
```

### Format Validation

```sql
-- Validate email format
SELECT 
    customer_id,
    email,
    CASE
        WHEN email NOT LIKE '%@%.%' THEN 'Invalid'
        WHEN LENGTH(email) > 100 THEN 'Too Long'
        WHEN POSITION('@' IN email) < 2 THEN 'Invalid'
        ELSE 'Valid'
    END AS email_status
FROM customers
WHERE email IS NOT NULL
  AND email NOT LIKE '%@%.%';

-- Validate phone format
SELECT 
    customer_id,
    phone,
    LENGTH(REGEXP_REPLACE(phone, '[^0-9]', '')) AS digit_count
FROM customers
WHERE phone IS NOT NULL
  AND LENGTH(REGEXP_REPLACE(phone, '[^0-9]', '')) != 10;

-- Validate date ranges
SELECT 
    customer_id,
    created_date,
    DATEDIFF(DAY, created_date, CURRENT_DATE()) AS days_old
FROM customers
WHERE created_date > CURRENT_DATE()  -- Future date (invalid)
   OR created_date < '1900-01-01'  -- Too old
   OR created_date IS NULL;
```

### Data Freshness Check

```sql
-- Monitor data freshness
SELECT 
    'customers' AS table_name,
    COUNT(*) AS record_count,
    MAX(updated_date) AS last_update,
    DATEDIFF(HOUR, MAX(updated_date), CURRENT_TIMESTAMP()) AS hours_since_update,
    CASE
        WHEN DATEDIFF(HOUR, MAX(updated_date), CURRENT_TIMESTAMP()) > 24 THEN 'STALE'
        WHEN DATEDIFF(HOUR, MAX(updated_date), CURRENT_TIMESTAMP()) > 12 THEN 'OLD'
        ELSE 'FRESH'
    END AS freshness_status
FROM customers;
```

## Complete Quality Monitoring Framework

```sql
-- Create quality metrics table
CREATE TABLE data_quality_metrics (
    check_id INT AUTOINCREMENT,
    table_name VARCHAR(100),
    check_name VARCHAR(100),
    check_timestamp TIMESTAMP,
    row_count INT,
    quality_score DECIMAL(5,2),
    issues_found INT,
    status VARCHAR(20)
);

-- Create monitoring stored procedure
CREATE OR REPLACE PROCEDURE check_data_quality()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    v_null_count INT;
    v_duplicate_count INT;
    v_total_count INT;
    v_quality_score DECIMAL(5,2);
BEGIN
    -- Count null values
    SELECT COUNT(*) INTO v_null_count
    FROM customers
    WHERE customer_id IS NULL OR name IS NULL;
    
    -- Count duplicates
    SELECT COUNT(*) INTO v_duplicate_count
    FROM (
        SELECT customer_id
        FROM customers
        GROUP BY customer_id
        HAVING COUNT(*) > 1
    );
    
    -- Total records
    SELECT COUNT(*) INTO v_total_count FROM customers;
    
    -- Calculate quality score
    v_quality_score := ROUND(100 - ((v_null_count + v_duplicate_count) * 100.0 / v_total_count), 2);
    
    -- Log results
    INSERT INTO data_quality_metrics (table_name, check_name, check_timestamp, row_count, quality_score, issues_found, status)
    VALUES ('customers', 'null_and_duplicate_check', CURRENT_TIMESTAMP(), v_total_count, v_quality_score, v_null_count + v_duplicate_count, 'COMPLETED');
    
    RETURN 'Quality Score: ' || v_quality_score || '%';
END;
$$;

-- Execute quality check
CALL check_data_quality();
```

### Create Quality Alerts

```sql
-- Create alerts for quality issues
CREATE TASK quality_check_task
  WAREHOUSE = analytics_wh
  SCHEDULE = 'USING CRON 0 1 * * * UTC'
AS
  INSERT INTO quality_alerts
  SELECT 
      CURRENT_TIMESTAMP(),
      'NULL_VALUES',
      COUNT(*) AS issue_count,
      'customers' AS affected_table
  FROM customers
  WHERE customer_id IS NULL OR name IS NULL OR email IS NULL
  HAVING COUNT(*) > 0;

ALTER TASK quality_check_task RESUME;
```

## Next Steps

1. **Automated Remediation:** Fix quality issues automatically
2. **Data Profiling:** Analyze data characteristics
3. **Quality Dashboards:** Visualize quality metrics

## Learning Outcomes

✅ Create null value checks  
✅ Detect duplicates  
✅ Validate formats  
✅ Monitor data freshness  
✅ Implement quality scoring  

---

**Last Updated:** February 18, 2026
