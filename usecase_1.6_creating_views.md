# Use Case 6: Creating Views for Simplified Access

## Problem Description

As your data warehouse grows, you need to:

1. Simplify complex queries for end users
2. Provide a consistent data interface
3. Hide underlying table complexity
4. Control which columns users can access
5. Pre-compute aggregations for performance
6. Maintain backward compatibility when tables change

Views are the solution to these challenges.

## Business Context

A company has complex customer and order tables. Different teams need:
- **Finance:** View showing revenue metrics
- **Marketing:** View showing customer segments
- **Sales:** View showing customer interactions
- **Executives:** Simplified dashboard view

Rather than each team writing complex queries, views provide consistent, secure access.

## Solution

### Step 1: Create a Simple View

```sql
-- Simple view showing active customers
CREATE VIEW active_customers AS
SELECT 
    customer_id,
    name,
    email,
    created_date
FROM customers
WHERE created_date >= DATEADD(MONTH, -12, CURRENT_DATE())
  AND status = 'ACTIVE';

-- Query the view like a table
SELECT * FROM active_customers;
```

### Step 2: Create a View with Joins

```sql
-- View combining customers and orders
CREATE VIEW customer_summary AS
SELECT 
    c.customer_id,
    c.name,
    c.country,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(o.order_amount), 0) AS lifetime_value,
    COALESCE(AVG(o.order_amount), 0) AS avg_order_value,
    MAX(o.order_date) AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.country;

-- Use the view
SELECT 
    customer_id,
    name,
    total_orders,
    lifetime_value
FROM customer_summary
WHERE lifetime_value > 1000
ORDER BY lifetime_value DESC;
```

### Step 3: Create a View with Transformations

```sql
-- View with calculated columns and categorization
CREATE VIEW customer_segments AS
SELECT 
    customer_id,
    name,
    email,
    created_date,
    DATEDIFF(MONTH, created_date, CURRENT_DATE()) AS months_as_customer,
    CASE
        WHEN DATEDIFF(MONTH, created_date, CURRENT_DATE()) < 3 THEN 'New'
        WHEN DATEDIFF(MONTH, created_date, CURRENT_DATE()) < 12 THEN 'Growing'
        ELSE 'Established'
    END AS customer_stage,
    CASE
        WHEN status = 'ACTIVE' THEN 'Active'
        WHEN status = 'INACTIVE' THEN 'Inactive'
        ELSE 'Unknown'
    END AS account_status
FROM customers;

-- Query segmentation view
SELECT 
    customer_stage,
    account_status,
    COUNT(*) AS customer_count
FROM customer_segments
GROUP BY customer_stage, account_status;
```

### Step 4: Create a View for Security (Column Restriction)

```sql
-- Sensitive customer data view (hides SSN, credit card)
CREATE VIEW public_customer_data AS
SELECT 
    customer_id,
    name,
    email,
    created_date,
    country,
    -- Exclude: ssn, credit_card, phone
FROM customers;

-- Grant access to this view instead of the base table
GRANT SELECT ON VIEW public_customer_data TO ROLE analyst_role;

-- Analysts can only see non-sensitive columns
SELECT * FROM public_customer_data;
```

### Step 5: Create a Materialized View for Performance

```sql
-- Materialized view for pre-computed aggregations
CREATE MATERIALIZED VIEW monthly_sales_summary AS
SELECT 
    DATE_TRUNC('MONTH', order_date) AS sales_month,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(*) AS total_orders,
    SUM(order_amount) AS monthly_revenue,
    AVG(order_amount) AS avg_order_value,
    STDDEV(order_amount) AS order_variance,
    MIN(order_amount) AS min_order,
    MAX(order_amount) AS max_order
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date);

-- Materialized view stores pre-computed results
-- Much faster than calculating each time
SELECT * FROM monthly_sales_summary;

-- Refresh when data changes
ALTER MATERIALIZED VIEW monthly_sales_summary REFRESH;
```

### Step 6: Create Dynamic Views with Parameters

```sql
-- Note: Snowflake views are static, but you can use procedures for dynamic behavior

-- Create view with filtering capability
CREATE VIEW top_customers AS
SELECT 
    customer_id,
    name,
    lifetime_value,
    RANK() OVER (ORDER BY lifetime_value DESC) AS customer_rank
FROM customer_summary
WHERE lifetime_value > 0;

-- Filter in query
SELECT * FROM top_customers WHERE customer_rank <= 10;
```

### Step 7: Create a View Hierarchy

```sql
-- Base view: Raw cleaned data
CREATE VIEW base_customers AS
SELECT * FROM customers WHERE status != 'DELETED';

-- Intermediate view: Enriched data
CREATE VIEW enriched_customers AS
SELECT 
    customer_id,
    name,
    email,
    country,
    DATEDIFF(YEAR, created_date, CURRENT_DATE()) AS tenure_years
FROM base_customers;

-- Final view: Business logic
CREATE VIEW vip_customers AS
SELECT 
    customer_id,
    name,
    email,
    tenure_years,
    'VIP' AS customer_type
FROM enriched_customers
WHERE tenure_years >= 3;

-- Query the highest-level view
SELECT * FROM vip_customers;
```

## Complete View Setup Script

```sql
-- Create comprehensive view structure

-- 1. Basic views
CREATE OR REPLACE VIEW active_customers AS
SELECT * FROM customers WHERE status = 'ACTIVE';

CREATE OR REPLACE VIEW new_customers AS
SELECT * FROM customers 
WHERE created_date >= DATEADD(MONTH, -3, CURRENT_DATE());

-- 2. Summary views
CREATE OR REPLACE VIEW customer_order_summary AS
SELECT 
    c.customer_id,
    c.name,
    COUNT(o.order_id) AS order_count,
    SUM(o.order_amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;

-- 3. Materialized views for performance
CREATE OR REPLACE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
    order_date,
    COUNT(*) AS order_count,
    SUM(order_amount) AS daily_revenue
FROM orders
GROUP BY order_date;

-- 4. Security views
CREATE OR REPLACE VIEW safe_customer_info AS
SELECT customer_id, name, email FROM customers;

-- Test views
SELECT * FROM active_customers LIMIT 10;
SELECT * FROM customer_order_summary WHERE order_count > 0;
SELECT * FROM daily_sales_summary ORDER BY order_date DESC LIMIT 30;
```

## View Management Operations

### Modify a View

```sql
-- Update view definition
CREATE OR REPLACE VIEW customer_summary AS
SELECT 
    c.customer_id,
    c.name,
    c.country,
    c.status,
    COUNT(o.order_id) AS total_orders,
    SUM(o.order_amount) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.country, c.status;
```

### View Information

```sql
-- Show all views in schema
SHOW VIEWS;

-- Show views like a pattern
SHOW VIEWS LIKE 'customer%';

-- Get view definition
SELECT GET_DDL('VIEW', 'customer_summary');

-- Show view dependencies
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE REFERENCED_OBJECT_NAME = 'CUSTOMER_SUMMARY';
```

### Drop a View

```sql
-- Remove a view
DROP VIEW customer_summary;

-- Drop only if exists
DROP VIEW IF EXISTS customer_summary;

-- Drop with cascade (removes dependent objects)
DROP VIEW customer_summary CASCADE;
```

## View Types Comparison

| Feature | View | Materialized View |
|---------|------|------------------|
| Storage | No - computed on query | Yes - stored data |
| Performance | Slower (recomputes) | Faster (pre-computed) |
| Freshness | Always current | Requires refresh |
| Update Overhead | None | Must refresh |
| Use Case | Real-time data | Summary aggregations |

## Advanced View Patterns

### Pattern 1: Slowly Changing Dimension View

```sql
-- View showing current state of slowly changing data
CREATE VIEW current_customer_status AS
SELECT 
    customer_id,
    name,
    email,
    status,
    valid_from,
    valid_to
FROM customer_history
WHERE valid_to = '9999-12-31'::DATE;
```

### Pattern 2: Time-Based View

```sql
-- View filtered to recent data
CREATE VIEW recent_transactions AS
SELECT * FROM transactions
WHERE transaction_date >= DATEADD(DAY, -30, CURRENT_DATE());
```

### Pattern 3: Aggregated Data View

```sql
-- View with pre-aggregated metrics
CREATE VIEW customer_metrics AS
SELECT 
    customer_id,
    COUNT(DISTINCT order_id) AS purchase_count,
    SUM(order_amount) AS total_revenue,
    AVG(order_amount) AS average_order,
    DATEDIFF(DAY, MIN(order_date), MAX(order_date)) AS customer_lifespan
FROM orders
GROUP BY customer_id;
```

### Pattern 4: Masked Data View

```sql
-- View that masks sensitive information
CREATE VIEW anonymized_customers AS
SELECT 
    HASH(customer_id) AS customer_hash,
    SUBSTRING(name, 1, 1) || '***' AS name_masked,
    '***@***.***' AS email_masked,
    country
FROM customers;
```

## View Query Examples

### Example 1: Sales Dashboard View

```sql
CREATE OR REPLACE VIEW sales_dashboard AS
SELECT 
    DATE_TRUNC('MONTH', o.order_date) AS month,
    c.country,
    COUNT(*) AS order_count,
    SUM(o.order_amount) AS revenue,
    AVG(o.order_amount) AS avg_order,
    COUNT(DISTINCT c.customer_id) AS customer_count
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY DATE_TRUNC('MONTH', o.order_date), c.country;

-- Use dashboard view
SELECT * FROM sales_dashboard 
WHERE month >= DATEADD(MONTH, -12, CURRENT_DATE())
ORDER BY month DESC, revenue DESC;
```

### Example 2: Customer Health View

```sql
CREATE OR REPLACE VIEW customer_health_score AS
WITH customer_metrics AS (
    SELECT 
        customer_id,
        COUNT(*) AS purchase_frequency,
        SUM(order_amount) AS monetary_value,
        DATEDIFF(DAY, MAX(order_date), CURRENT_DATE()) AS days_since_purchase
    FROM orders
    GROUP BY customer_id
)
SELECT 
    c.customer_id,
    c.name,
    cm.purchase_frequency,
    cm.monetary_value,
    cm.days_since_purchase,
    CASE
        WHEN cm.purchase_frequency >= 10 AND cm.days_since_purchase < 30 THEN 'Healthy'
        WHEN cm.purchase_frequency >= 5 AND cm.days_since_purchase < 90 THEN 'At-Risk'
        WHEN cm.days_since_purchase > 180 THEN 'Dormant'
        ELSE 'Inactive'
    END AS health_status
FROM customers c
LEFT JOIN customer_metrics cm ON c.customer_id = cm.customer_id;
```

## Best Practices

### 1. Use Descriptive Names

```sql
-- Good: Clear purpose
CREATE VIEW customer_annual_summary AS ...

-- Bad: Vague
CREATE VIEW data_view AS ...
```

### 2. Document Views

```sql
CREATE VIEW customer_annual_summary AS
-- Purpose: Summarize customer spending by year for reporting
-- Last Updated: 2026-02-18
-- Owner: Analytics Team
SELECT ...;
```

### 3. Create View Hierarchy

```sql
-- Atomic views (based on tables)
CREATE VIEW customers_active AS SELECT * FROM customers WHERE status = 'ACTIVE';

-- Composite views (based on atomic views)
CREATE VIEW customers_summary AS 
SELECT * FROM customers_active c 
LEFT JOIN orders o ON c.customer_id = o.customer_id;

-- Analytical views (business logic)
CREATE VIEW customer_segments AS 
SELECT * FROM customers_summary WHERE ...;
```

### 4. Use Materialized Views for Complex Aggregations

```sql
-- If view takes > 1 second to compute, consider materialization
CREATE MATERIALIZED VIEW slow_aggregation AS
SELECT 
    DATE_TRUNC('DAY', order_date),
    SUM(order_amount),
    COUNT(*),
    ...massive aggregation...
FROM large_table
GROUP BY DATE_TRUNC('DAY', order_date);
```

## Performance Considerations

### View Performance Impact

```sql
-- Views with no aggregation: minimal overhead
CREATE VIEW fast_view AS SELECT * FROM table WHERE condition;

-- Views with joins: moderate overhead
CREATE VIEW medium_view AS SELECT * FROM t1 JOIN t2;

-- Views with aggregation: may be slow (use materialized view)
CREATE VIEW slow_view AS SELECT ... FROM huge_table GROUP BY ...;
```

### Refresh Materialized View Strategy

```sql
-- Create task to refresh materialized view daily
CREATE TASK refresh_sales_summary
  WAREHOUSE = compute_wh
  SCHEDULE = 'USING CRON 0 2 * * * UTC'
AS
  ALTER MATERIALIZED VIEW monthly_sales_summary REFRESH;

-- Enable task
ALTER TASK refresh_sales_summary RESUME;
```

## Security with Views

```sql
-- Grant view access instead of table access
REVOKE SELECT ON TABLE customers FROM ROLE analyst_role;
GRANT SELECT ON VIEW safe_customer_info TO ROLE analyst_role;

-- Users can only see what's in the view
SELECT * FROM safe_customer_info; -- works
SELECT ssn FROM customers; -- error - no access
```

## Next Steps

1. **Implement security views** for different user roles
2. **Create materialized views** for performance
3. **Build view hierarchy** for complex data models
4. **Monitor view usage** and refresh patterns

## Learning Outcomes

✅ Create simple and complex views  
✅ Use materialized views for performance  
✅ Implement security with views  
✅ Manage view lifecycle  
✅ Build view hierarchies  

## Related Use Cases

- **Use Case 1:** Creating Your First Database and Table
- **Use Case 3:** Basic SELECT Queries
- **Use Case 4:** Using Snowflake Roles and Users
- **Use Case 23:** Creating Materialized Views for Performance

---

**Last Updated:** February 18, 2026
