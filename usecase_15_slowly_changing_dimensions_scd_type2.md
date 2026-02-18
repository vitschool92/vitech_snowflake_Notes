# Use Case 15: Implementing Slowly Changing Dimensions (SCD Type 2)

## Problem Description

Data warehouses must track historical changes. Slowly Changing Dimensions Type 2 allows you to:

1. Keep historical versions of data
2. Track when changes occur
3. Query data as of a specific point in time
4. Support "as of" analysis
5. Maintain data lineage

## Business Context

A company needs to track:
- When customer information changes
- Historical product prices
- Employee department changes
- All historical versions for auditing

## Solution

### SCD Type 2 Structure

```sql
-- Create SCD Type 2 dimension table
CREATE TABLE customer_dimension_scd (
    customer_key INT AUTOINCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    name VARCHAR(100),
    email VARCHAR(100),
    country VARCHAR(50),
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL,
    is_current BOOLEAN NOT NULL,
    created_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    updated_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Insert initial version of customer
INSERT INTO customer_dimension_scd 
(customer_id, name, email, country, valid_from, valid_to, is_current)
VALUES
(101, 'John Smith', 'john@example.com', 'USA', '2025-01-01', '9999-12-31'::DATE, TRUE),
(102, 'Sarah Johnson', 'sarah@example.com', 'Canada', '2025-01-01', '9999-12-31'::DATE, TRUE);
```

### Handling Customer Changes

```sql
-- When customer data changes (email update):

-- 1. Mark old record as no longer current
UPDATE customer_dimension_scd
SET valid_to = DATEADD(DAY, -1, CURRENT_DATE()),
    is_current = FALSE,
    updated_timestamp = CURRENT_TIMESTAMP()
WHERE customer_id = 101
  AND is_current = TRUE;

-- 2. Insert new version
INSERT INTO customer_dimension_scd
(customer_id, name, email, country, valid_from, valid_to, is_current)
VALUES
(101, 'John Smith', 'john.smith@newdomain.com', 'USA', CURRENT_DATE(), '9999-12-31'::DATE, TRUE);
```

### Querying Historical Data

```sql
-- Current customer information
SELECT * FROM customer_dimension_scd
WHERE customer_id = 101
  AND is_current = TRUE;

-- All historical versions
SELECT * FROM customer_dimension_scd
WHERE customer_id = 101
ORDER BY valid_from;

-- Customer state on specific date
SELECT * FROM customer_dimension_scd
WHERE customer_id = 101
  AND valid_from <= '2025-02-01'
  AND valid_to >= '2025-02-01';

-- Customer state as of 30 days ago
SELECT * FROM customer_dimension_scd
WHERE customer_id = 101
  AND valid_from <= DATEADD(DAY, -30, CURRENT_DATE())
  AND valid_to >= DATEADD(DAY, -30, CURRENT_DATE());
```

## Complete SCD Type 2 Implementation

```sql
-- 1. Create staging table
CREATE TABLE customer_staging (
    customer_id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    country VARCHAR(50),
    load_date DATE
);

-- Insert source data
INSERT INTO customer_staging VALUES
(101, 'John Smith', 'john.smith@example.com', 'USA', CURRENT_DATE()),
(102, 'Sarah Johnson', 'sarah.johnson@example.com', 'Canada', CURRENT_DATE());

-- 2. Create SCD merge procedure
CREATE OR REPLACE PROCEDURE merge_customer_scd()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    v_message VARCHAR;
BEGIN
    -- Close old records where data changed
    UPDATE customer_dimension_scd dim
    SET valid_to = DATEADD(DAY, -1, CURRENT_DATE()),
        is_current = FALSE
    WHERE is_current = TRUE
      AND EXISTS (
        SELECT 1 FROM customer_staging stg
        WHERE stg.customer_id = dim.customer_id
          AND (
            stg.name != dim.name
            OR stg.email != dim.email
            OR stg.country != dim.country
          )
      );
    
    -- Insert new records for changed/new customers
    INSERT INTO customer_dimension_scd
    (customer_id, name, email, country, valid_from, valid_to, is_current)
    SELECT 
        stg.customer_id,
        stg.name,
        stg.email,
        stg.country,
        CURRENT_DATE(),
        '9999-12-31'::DATE,
        TRUE
    FROM customer_staging stg
    WHERE NOT EXISTS (
        SELECT 1 FROM customer_dimension_scd dim
        WHERE dim.customer_id = stg.customer_id
          AND dim.is_current = TRUE
          AND stg.name = dim.name
          AND stg.email = dim.email
          AND stg.country = dim.country
    );
    
    v_message := 'SCD merge completed successfully';
    RETURN v_message;
END;
$$;

-- Execute merge
CALL merge_customer_scd();

-- 3. Create view for current state
CREATE VIEW customer_current_state AS
SELECT 
    customer_id,
    name,
    email,
    country
FROM customer_dimension_scd
WHERE is_current = TRUE;

-- 4. Create view for historical audit
CREATE VIEW customer_change_history AS
SELECT 
    customer_id,
    name,
    email,
    country,
    valid_from,
    valid_to,
    DATEDIFF(DAY, valid_from, valid_to) AS days_valid
FROM customer_dimension_scd
ORDER BY customer_id, valid_from;
```

## SCD Type 2 Patterns

### Pattern 1: Identify Changed Records

```sql
-- Find all changes for a customer
SELECT 
    customer_id,
    valid_from,
    valid_to,
    name,
    email,
    country,
    LAG(name) OVER (PARTITION BY customer_id ORDER BY valid_from) AS prev_name,
    LAG(email) OVER (PARTITION BY customer_id ORDER BY valid_from) AS prev_email,
    LAG(country) OVER (PARTITION BY customer_id ORDER BY valid_from) AS prev_country
FROM customer_dimension_scd
WHERE customer_id = 101
ORDER BY valid_from;
```

### Pattern 2: Change Timeline

```sql
-- Timeline of changes
SELECT 
    customer_id,
    valid_from,
    valid_to,
    CASE
        WHEN LAG(name) OVER (PARTITION BY customer_id ORDER BY valid_from) IS NULL THEN 'New Record'
        WHEN LAG(name) OVER (PARTITION BY customer_id ORDER BY valid_from) != name THEN 'Name Changed'
        WHEN LAG(email) OVER (PARTITION BY customer_id ORDER BY valid_from) != email THEN 'Email Changed'
        WHEN LAG(country) OVER (PARTITION BY customer_id ORDER BY valid_from) != country THEN 'Country Changed'
    END AS change_type
FROM customer_dimension_scd
ORDER BY customer_id, valid_from;
```

### Pattern 3: Audit Trail

```sql
-- Complete audit trail
CREATE TABLE customer_audit_trail (
    audit_id INT AUTOINCREMENT,
    customer_id INT,
    change_date DATE,
    change_type VARCHAR,
    old_value VARCHAR,
    new_value VARCHAR,
    changed_by VARCHAR DEFAULT CURRENT_USER(),
    changed_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Log changes
INSERT INTO customer_audit_trail (customer_id, change_date, change_type, old_value, new_value)
SELECT 
    customer_id,
    valid_from,
    'Email Change',
    LAG(email) OVER (PARTITION BY customer_id ORDER BY valid_from),
    email
FROM customer_dimension_scd
WHERE LAG(email) OVER (PARTITION BY customer_id ORDER BY valid_from) != email;
```

## Next Steps

1. **Fact Table Joins:** Join SCD dimensions with fact tables
2. **Temporal Queries:** Analyze data across time periods
3. **Change Detection:** Automated change detection

## Learning Outcomes

✅ Understand SCD Type 2 concept  
✅ Implement dimension versioning  
✅ Track historical changes  
✅ Query temporal data  
✅ Build audit trails  

## Related Use Cases

- **Use Case 26:** Building Data Marts with Dimensional Modeling
- **Use Case 17:** Time Travel and Data Recovery

---

**Last Updated:** February 18, 2026
