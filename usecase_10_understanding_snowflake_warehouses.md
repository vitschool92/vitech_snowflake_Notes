# Use Case 10: Understanding Snowflake Warehouses

## Problem Description

Snowflake's compute power is provided by warehouses. You need to:

1. Create and configure warehouses
2. Understand warehouse sizing
3. Manage warehouse performance
4. Implement auto-suspend and auto-resume
5. Monitor warehouse usage and costs
6. Scale warehouses based on needs

## Business Context

A company has different workloads:
- **Data Loading:** Requires fast compute (4-8 hour daily load)
- **Analytics Queries:** Moderate compute (business hours)
- **Real-time Dashboards:** Always available
- **Ad-hoc Analysis:** Occasional heavy queries

Different workloads need different warehouse configurations.

## Solution

### Create Basic Warehouse

```sql
-- Create a simple warehouse
CREATE WAREHOUSE compute_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  COMMENT = 'General purpose compute warehouse';

-- Verify warehouse creation
SHOW WAREHOUSES;

-- Describe warehouse
DESCRIBE WAREHOUSE compute_wh;
```

### Warehouse Sizing Options

```sql
-- Warehouse sizes (from smallest to largest)
-- XSMALL: 1 credit/hour - for small queries
-- SMALL: 2 credits/hour
-- MEDIUM: 4 credits/hour (balanced)
-- LARGE: 8 credits/hour
-- XLARGE: 16 credits/hour
-- XXLARGE: 32 credits/hour - for heavy processing
-- XXXLARGE: 64 credits/hour
-- 4XLARGE: 128 credits/hour - for massive parallel processing

-- Create warehouse with specific size
CREATE WAREHOUSE analytics_wh
  WAREHOUSE_SIZE = 'SMALL'
  COMMENT = 'For analytics queries';

CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'LARGE'
  COMMENT = 'For ETL processes';

CREATE WAREHOUSE reporting_wh
  WAREHOUSE_SIZE = 'XSMALL'
  COMMENT = 'For lightweight reports';
```

### Configure Auto-Suspend and Auto-Resume

```sql
-- Create warehouse with auto-suspend
CREATE WAREHOUSE intelligent_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 600  -- Suspend after 600 seconds (10 minutes) of inactivity
  AUTO_RESUME = TRUE  -- Automatically resume when needed
  COMMENT = 'Auto-suspend after 10 minutes';

-- Modify existing warehouse settings
ALTER WAREHOUSE compute_wh SET AUTO_SUSPEND = 300;  -- 5 minutes
ALTER WAREHOUSE compute_wh SET AUTO_RESUME = TRUE;

-- Disable auto-suspend (runs continuously)
ALTER WAREHOUSE compute_wh SET AUTO_SUSPEND = NULL;
```

### Scaling Warehouses

```sql
-- Scale up a warehouse (increase size)
ALTER WAREHOUSE compute_wh SET WAREHOUSE_SIZE = 'LARGE';

-- Scale down a warehouse (decrease size)
ALTER WAREHOUSE compute_wh SET WAREHOUSE_SIZE = 'SMALL';

-- Note: Scaling operations are immediate for new queries
-- Current queries continue with old size
```

### Multi-Cluster Warehouses

```sql
-- Create multi-cluster warehouse for high availability
CREATE WAREHOUSE multi_cluster_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  MAX_CLUSTER_COUNT = 3    -- Can scale to 3 clusters
  MIN_CLUSTER_COUNT = 1    -- Minimum 1 cluster
  SCALING_POLICY = 'STANDARD'  -- Add clusters when queue forms
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  COMMENT = 'Multi-cluster warehouse for high throughput';

-- View cluster details
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE WAREHOUSE_NAME = 'MULTI_CLUSTER_WH'
ORDER BY START_TIME DESC
LIMIT 20;
```

### Warehouse Selection and Usage

```sql
-- Set default warehouse for session
USE WAREHOUSE compute_wh;

-- Check current warehouse
SELECT CURRENT_WAREHOUSE();

-- Execute query on specific warehouse (override default)
SELECT * FROM customers;  -- Uses current warehouse

-- Switch warehouses mid-session
USE WAREHOUSE analytics_wh;
SELECT * FROM orders;

-- Switch back
USE WAREHOUSE compute_wh;
```

## Complete Warehouse Configuration Script

```sql
-- Comprehensive warehouse setup

-- 1. Create warehouses for different workloads
CREATE OR REPLACE WAREHOUSE loading_wh
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  COMMENT = 'For data loading operations';

CREATE OR REPLACE WAREHOUSE analytics_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 600
  AUTO_RESUME = TRUE
  COMMENT = 'For analytical queries';

CREATE OR REPLACE WAREHOUSE reporting_wh
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 900
  AUTO_RESUME = TRUE
  COMMENT = 'For standard reports';

CREATE OR REPLACE WAREHOUSE dev_wh
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  COMMENT = 'For development and testing';

-- 2. Set default warehouse for current session
USE WAREHOUSE analytics_wh;

-- 3. Verify all warehouses
SHOW WAREHOUSES;
```

## Warehouse State Management

### Suspend/Resume Warehouse

```sql
-- Manually suspend a warehouse (stop it)
ALTER WAREHOUSE compute_wh SUSPEND;

-- Manually resume a warehouse (start it)
ALTER WAREHOUSE compute_wh RESUME;

-- Check warehouse state
SHOW WAREHOUSES LIKE 'compute_wh';
-- State column shows: RUNNING or SUSPENDED

-- Drop a warehouse
DROP WAREHOUSE compute_wh;
```

## Monitoring Warehouse Performance

### View Warehouse History

```sql
-- Query warehouse metering history (credit consumption)
SELECT 
    WAREHOUSE_NAME,
    START_TIME,
    END_TIME,
    CREDITS_USED,
    CREDITS_USED_COMPUTE,
    CREDITS_USED_CLOUD_SERVICES
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(DAY, -7, CURRENT_DATE())
ORDER BY START_TIME DESC
LIMIT 100;

-- Credits by warehouse (cost analysis)
SELECT 
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS total_credits,
    COUNT(*) AS query_count,
    AVG(CREDITS_USED) AS avg_credits_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(MONTH, -1, CURRENT_DATE())
GROUP BY WAREHOUSE_NAME
ORDER BY total_credits DESC;
```

### Query Performance by Warehouse

```sql
-- View query execution statistics
SELECT 
    WAREHOUSE_NAME,
    COUNT(*) AS query_count,
    AVG(EXECUTION_TIME) AS avg_execution_time_ms,
    MAX(EXECUTION_TIME) AS max_execution_time_ms,
    SUM(BYTES_SCANNED) AS total_bytes_scanned,
    AVG(BYTES_SCANNED) AS avg_bytes_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE EXECUTION_DATE >= DATEADD(DAY, -7, CURRENT_DATE())
  AND WAREHOUSE_NAME IS NOT NULL
GROUP BY WAREHOUSE_NAME
ORDER BY query_count DESC;
```

### Identify Long-Running Queries

```sql
-- Find slow queries by warehouse
SELECT 
    QUERY_ID,
    USER_NAME,
    WAREHOUSE_NAME,
    QUERY_TEXT,
    EXECUTION_TIME / 1000 AS execution_seconds,
    BYTES_SCANNED,
    BYTES_WRITTEN
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE EXECUTION_DATE >= DATEADD(DAY, -7, CURRENT_DATE())
  AND WAREHOUSE_NAME = 'ANALYTICS_WH'
ORDER BY EXECUTION_TIME DESC
LIMIT 50;
```

## Cost Management

### Warehouse-Based Cost Analysis

```sql
-- Calculate monthly costs (assuming $4 per credit)
SELECT 
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS total_credits_used,
    SUM(CREDITS_USED) * 4.0 AS estimated_cost_usd,
    COUNT(DISTINCT DATE(END_TIME)) AS active_days,
    AVG(CREDITS_USED) AS avg_daily_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(MONTH, -1, CURRENT_DATE())
GROUP BY WAREHOUSE_NAME
ORDER BY total_credits_used DESC;

-- Hourly credit usage pattern
SELECT 
    WAREHOUSE_NAME,
    HOUR(START_TIME) AS hour_of_day,
    AVG(CREDITS_USED) AS avg_credits_per_hour,
    COUNT(*) AS query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(MONTH, -1, CURRENT_DATE())
GROUP BY WAREHOUSE_NAME, HOUR(START_TIME)
ORDER BY WAREHOUSE_NAME, hour_of_day;
```

## Right-Sizing Warehouses

### Analysis for Warehouse Optimization

```sql
-- Find underutilized warehouses
SELECT 
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS total_credits,
    COUNT(DISTINCT DATE(END_TIME)) AS active_days,
    ROUND(SUM(CREDITS_USED) / 30.0, 2) AS avg_daily_credits,
    CASE
        WHEN SUM(CREDITS_USED) < 50 THEN 'Downsize or remove'
        WHEN SUM(CREDITS_USED) < 200 THEN 'Consider downsizing'
        WHEN SUM(CREDITS_USED) > 5000 THEN 'Consider upsizing'
        ELSE 'Right-sized'
    END AS recommendation
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(MONTH, -3, CURRENT_DATE())
GROUP BY WAREHOUSE_NAME;

-- Peak vs average usage
SELECT 
    WAREHOUSE_NAME,
    MIN(CREDITS_USED) AS min_credits_in_period,
    AVG(CREDITS_USED) AS avg_credits_per_period,
    MAX(CREDITS_USED) AS max_credits_in_period,
    STDDEV(CREDITS_USED) AS credit_variance
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(MONTH, -1, CURRENT_DATE())
GROUP BY WAREHOUSE_NAME;
```

## Warehouse Best Practices

### 1. Separate Workloads

```sql
-- Good: Separate warehouses for different workloads
-- Loading operations → loading_wh (LARGE, always ready)
-- Analytics queries → analytics_wh (MEDIUM, auto-suspend)
-- Reports → reporting_wh (SMALL, auto-suspend)

-- Bad: Single warehouse for all workloads
-- Can cause resource contention and cost inefficiency
```

### 2. Enable Auto-Suspend

```sql
-- Good: Auto-suspend after inactivity
ALTER WAREHOUSE analytics_wh SET AUTO_SUSPEND = 600;

-- Bad: Always running (wastes credits)
ALTER WAREHOUSE analytics_wh SET AUTO_SUSPEND = NULL;
```

### 3. Use Appropriate Sizes

```sql
-- Match warehouse size to workload
-- Simple SELECT queries → XSMALL
-- Aggregations and joins → SMALL/MEDIUM
-- Complex analytics → LARGE/XLARGE
-- Data loading → LARGE/XLARGE
-- Real-time dashboards → MEDIUM (dedicated)
```

### 4. Monitor and Adjust

```sql
-- Regularly review warehouse utilization
-- Scale up if queries take too long
-- Scale down if resources are unused
-- Remove unused warehouses

-- Set up monitoring queries
-- Run weekly to identify optimization opportunities
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD(WEEK, -1, CURRENT_DATE())
  AND WAREHOUSE_NAME = 'YOUR_WAREHOUSE';
```

## Troubleshooting Warehouse Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Warehouse suspended" | Auto-suspend triggered | Query will resume warehouse automatically |
| "Not enough resources" | Warehouse too small | Scale up or use different warehouse |
| "Queue depth high" | Too many queries | Add more clusters or increase warehouse size |
| "Slow queries" | Warehouse size mismatch | Match size to query complexity |

## Next Steps

1. **Query Optimization:** Use appropriate warehouse sizes for queries
2. **Cost Management:** Monitor and optimize warehouse usage
3. **Multi-Cluster Setup:** Handle variable workloads

## Learning Outcomes

✅ Create and configure warehouses  
✅ Understand warehouse sizing  
✅ Implement auto-suspend/resume  
✅ Scale warehouses for performance  
✅ Monitor warehouse usage and costs  
✅ Right-size warehouses for workloads  

## Related Use Cases

- **Use Case 25:** Monitoring Queries and Resource Usage
- **Use Case 35:** Dynamic Data Warehouse Scaling

---

**Last Updated:** February 18, 2026
