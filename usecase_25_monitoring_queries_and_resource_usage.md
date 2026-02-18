# Use Case 25: Monitoring Queries and Resource Usage

## Problem Description

Query monitoring enables you to:

1. Track execution times
2. Monitor resource consumption
3. Identify slow queries
4. Optimize expensive operations
5. Control costs

## Business Context

A company needs to:
- Optimize slow queries
- Understand credit usage
- Identify resource-heavy operations
- Set up cost alerts

## Solution

```sql
-- Query execution history
SELECT 
    QUERY_ID,
    USER_NAME,
    WAREHOUSE_NAME,
    QUERY_TEXT,
    EXECUTION_TIME / 1000 AS execution_seconds,
    BYTES_SCANNED,
    BYTES_WRITTEN,
    CREDITS_USED
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE EXECUTION_DATE >= DATEADD(DAY, -7, CURRENT_DATE())
ORDER BY EXECUTION_TIME DESC
LIMIT 50;

-- Warehouse credit usage
SELECT 
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS total_credits,
    COUNT(*) AS query_count,
    AVG(CREDITS_USED) AS avg_credits,
    SUM(CREDITS_USED) * 4 AS estimated_cost
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(MONTH, -1, CURRENT_DATE())
GROUP BY WAREHOUSE_NAME
ORDER BY total_credits DESC;

-- Top resource consumers
SELECT 
    USER_NAME,
    SUM(CREDITS_USED) AS total_credits,
    COUNT(*) AS query_count,
    AVG(EXECUTION_TIME / 1000) AS avg_execution_seconds
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE EXECUTION_DATE >= DATEADD(DAY, -30, CURRENT_DATE())
GROUP BY USER_NAME
ORDER BY total_credits DESC
LIMIT 20;

-- Create monitoring task
CREATE TASK monitor_slow_queries
  WAREHOUSE = analytics_wh
  SCHEDULE = 'USING CRON 0 */6 * * * UTC'
AS
  INSERT INTO slow_query_log
  SELECT 
      QUERY_ID,
      USER_NAME,
      WAREHOUSE_NAME,
      QUERY_TEXT,
      EXECUTION_TIME / 1000 AS execution_seconds,
      CURRENT_TIMESTAMP() AS captured_at
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE EXECUTION_TIME > 300000  -- Queries > 5 minutes
    AND EXECUTION_DATE >= DATEADD(HOUR, -6, CURRENT_DATE());

ALTER TASK monitor_slow_queries RESUME;
```

## Query Performance Optimization

```sql
-- Analyze query execution plan
EXPLAIN
SELECT * FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.status = 'ACTIVE';

-- Get query profile
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Identify scans vs seeks
SELECT 
    COUNT(*) AS query_count,
    SUM(BYTES_SCANNED) AS total_bytes_scanned,
    AVG(BYTES_SCANNED) AS avg_bytes_per_query,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY BYTES_SCANNED) AS p95_bytes
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE EXECUTION_DATE >= DATEADD(DAY, -7, CURRENT_DATE())
  AND QUERY_TEXT LIKE '%customers%';
```

## Next Steps

1. **Query Optimization:** Apply optimization techniques
2. **Cost Management:** Implement cost controls
3. **Alerting:** Set up cost alerts

## Learning Outcomes

✅ Monitor query execution  
✅ Track resource usage  
✅ Identify slow queries  
✅ Calculate query costs  
✅ Optimize expensive queries  

## Related Use Cases

- **Use Case 10:** Understanding Snowflake Warehouses
- **Use Case 42:** Advanced Query Optimization

---

**Last Updated:** February 18, 2026
