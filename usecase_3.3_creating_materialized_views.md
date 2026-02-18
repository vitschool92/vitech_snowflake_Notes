# Use Case 23: Creating Materialized Views for Performance

## Problem Description

Materialized Views enable you to:

1. Pre-compute aggregations
2. Store results physically
3. Improve query speed
4. Reduce compute costs
5. Refresh on schedule

## Business Context

Complex aggregations run frequently. Materializing them improves performance dramatically.

## Solution

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW customer_metrics AS
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(order_amount) AS total_spent,
    AVG(order_amount) AS avg_order,
    MAX(order_date) AS last_order_date
FROM orders
GROUP BY customer_id;

-- Query view (uses pre-computed results)
SELECT * FROM customer_metrics
WHERE total_spent > 1000
ORDER BY total_spent DESC;

-- Refresh view
ALTER MATERIALIZED VIEW customer_metrics REFRESH;

-- Automatic refresh with task
CREATE TASK refresh_customer_metrics
  WAREHOUSE = analytics_wh
  SCHEDULE = 'USING CRON 0 2 * * * UTC'
AS
  ALTER MATERIALIZED VIEW customer_metrics REFRESH;

ALTER TASK refresh_customer_metrics RESUME;

-- Monitor materialized view
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE NAME = 'CUSTOMER_METRICS'
ORDER BY START_TIME DESC
LIMIT 10;
```

## Next Steps

1. **Incremental Refresh:** Only refresh changed data
2. **Dynamic Materialization:** Automatically decide what to materialize
3. **Cost Analysis:** Compare materialized vs dynamic views

## Learning Outcomes

✅ Create materialized views  
✅ Refresh views  
✅ Schedule automatic refresh  
✅ Monitor performance  

---

**Last Updated:** February 18, 2026
