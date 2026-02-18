# Use Case 17: Time Travel and Data Recovery

## Problem Description

Snowflake's Time Travel feature enables you to:

1. Query historical versions of data
2. Recover deleted data
3. Restore tables to previous states
4. Audit data changes
5. Implement "as of" queries

## Business Context

A company accidentally deleted customer records or modified important data. Time Travel allows recovery without backup restore.

## Solution

### Query Historical Data

```sql
-- Query data from 1 hour ago (OFFSET in seconds)
SELECT * FROM customers
AT (OFFSET => -3600);

-- Query from 1 day ago
SELECT * FROM customers
AT (OFFSET => -86400);

-- Query from specific timestamp
SELECT * FROM customers
BEFORE (TIMESTAMP => '2026-02-17 10:00:00'::TIMESTAMP_NTZ);

-- Query specific statement
SELECT * FROM customers
BEFORE (STATEMENT => 'abc123xyz');  -- Query ID
```

### Restore Deleted Data

```sql
-- Restore entire table
UNDROP TABLE customers;

-- Restore dropped schema
UNDROP SCHEMA public;

-- Restore dropped database
UNDROP DATABASE customer_db;

-- Check if object existed
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.DROPPED_OBJECTS_HISTORY
WHERE OBJECT_NAME = 'CUSTOMERS'
ORDER BY DROPPED_ON DESC;
```

## Time Travel Retention

```sql
-- Check current retention (default 1 day)
SHOW TABLES LIKE 'customers';
-- RETENTION_TIME column shows days

-- Increase retention (requires Enterprise edition)
ALTER TABLE customers SET DATA_RETENTION_TIME_IN_DAYS = 90;

-- Schema level retention
ALTER SCHEMA public SET DATA_RETENTION_TIME_IN_DAYS = 30;

-- Database level
ALTER DATABASE customer_db SET DATA_RETENTION_TIME_IN_DAYS = 60;
```

## Recovery Patterns

```sql
-- Find and recover specific data
CREATE TABLE customers_recovered AS
SELECT * FROM customers
BEFORE (TIMESTAMP => DATEADD(DAY, -1, CURRENT_TIMESTAMP()));

-- Compare before/after
WITH deleted_records AS (
    SELECT * FROM customers
    AT (OFFSET => -86400)
),
current_records AS (
    SELECT * FROM customers
)
SELECT * FROM deleted_records
EXCEPT
SELECT * FROM current_records;

-- Recover specific rows
INSERT INTO customers
SELECT * FROM customers
BEFORE (TIMESTAMP => '2026-02-17 10:00:00'::TIMESTAMP_NTZ)
WHERE customer_id = 101
  AND customer_id NOT IN (SELECT customer_id FROM customers);
```

## Next Steps

1. **Automated Recovery:** Implement automated recovery procedures
2. **Audit Trails:** Track all changes with Time Travel
3. **Data Governance:** Use for compliance audits

## Learning Outcomes

✅ Query historical data  
✅ Restore deleted objects  
✅ Configure retention periods  
✅ Implement recovery procedures  

---

**Last Updated:** February 18, 2026
