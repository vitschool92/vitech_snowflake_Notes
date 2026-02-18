# Use Case 7: Working with Dates and Time Functions

## Problem Description

Most data warehouse queries involve time-based analysis. You need to:

1. Extract date components (year, month, day)
2. Calculate date differences
3. Add/subtract intervals from dates
4. Format dates for reporting
5. Handle timezones
6. Create time-based aggregations

## Business Context

An e-commerce company needs to:
- Analyze sales by month/quarter/year
- Calculate customer tenure
- Identify seasonal trends
- Track order fulfillment times
- Generate time-based reports

## Solution

### Date Component Extraction

```sql
-- Extract parts of a date
SELECT 
    order_date,
    YEAR(order_date) AS year,
    QUARTER(order_date) AS quarter,
    MONTH(order_date) AS month,
    DAY(order_date) AS day,
    WEEK(order_date) AS week_number,
    DAYOFWEEK(order_date) AS day_of_week,
    DAYOFYEAR(order_date) AS day_of_year
FROM orders
LIMIT 5;
```

### Date Arithmetic

```sql
-- Calculate differences between dates
SELECT 
    order_date,
    CURRENT_DATE() AS today,
    DATEDIFF(DAY, order_date, CURRENT_DATE()) AS days_since_order,
    DATEDIFF(MONTH, order_date, CURRENT_DATE()) AS months_since_order,
    DATEDIFF(YEAR, order_date, CURRENT_DATE()) AS years_since_order
FROM orders
LIMIT 10;

-- Add/subtract date intervals
SELECT 
    order_date,
    DATEADD(DAY, 7, order_date) AS delivery_date_7days,
    DATEADD(MONTH, 1, order_date) AS next_month,
    DATEADD(YEAR, 1, order_date) AS one_year_later
FROM orders
LIMIT 10;
```

### Date Truncation and Rounding

```sql
-- Truncate dates to specific intervals
SELECT 
    order_date,
    DATE_TRUNC('YEAR', order_date) AS year_start,
    DATE_TRUNC('QUARTER', order_date) AS quarter_start,
    DATE_TRUNC('MONTH', order_date) AS month_start,
    DATE_TRUNC('WEEK', order_date) AS week_start,
    DATE_TRUNC('DAY', order_date) AS day_start
FROM orders
LIMIT 5;

-- Group sales by month
SELECT 
    DATE_TRUNC('MONTH', order_date) AS sales_month,
    COUNT(*) AS order_count,
    SUM(order_amount) AS monthly_revenue
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date)
ORDER BY sales_month DESC;
```

### Timestamp Functions

```sql
-- Work with timestamps (date + time)
SELECT 
    order_timestamp,
    DATE(order_timestamp) AS order_date,
    TIME(order_timestamp) AS order_time,
    HOUR(order_timestamp) AS hour,
    MINUTE(order_timestamp) AS minute,
    SECOND(order_timestamp) AS second
FROM orders
WHERE order_timestamp IS NOT NULL
LIMIT 10;

-- Calculate time differences
SELECT 
    order_timestamp,
    delivered_timestamp,
    DATEDIFF(HOUR, order_timestamp, delivered_timestamp) AS delivery_hours,
    DATEDIFF(MINUTE, order_timestamp, delivered_timestamp) AS delivery_minutes
FROM orders
WHERE delivered_timestamp IS NOT NULL;
```

### Timezone Handling

```sql
-- Convert timestamps to different timezones
SELECT 
    order_timestamp AT TIME ZONE 'UTC' AS utc_time,
    order_timestamp AT TIME ZONE 'America/New_York' AS eastern_time,
    order_timestamp AT TIME ZONE 'America/Los_Angeles' AS pacific_time,
    order_timestamp AT TIME ZONE 'Europe/London' AS london_time
FROM orders
LIMIT 10;

-- Convert to UTC
SELECT 
    order_timestamp,
    CONVERT_TIMEZONE('America/New_York', 'UTC', order_timestamp) AS utc_time
FROM orders
LIMIT 10;
```

### String to Date Conversion

```sql
-- Parse date strings
SELECT 
    order_date_string,
    TO_DATE(order_date_string, 'YYYY-MM-DD') AS parsed_date,
    TO_DATE(order_date_string, 'DD/MM/YYYY') AS alternate_format
FROM orders_raw
LIMIT 10;

-- Parse timestamps
SELECT 
    order_timestamp_string,
    TO_TIMESTAMP(order_timestamp_string, 'YYYY-MM-DD HH:MI:SS') AS parsed_timestamp
FROM orders_raw
LIMIT 10;
```

### Date to String Formatting

```sql
-- Format dates for reporting
SELECT 
    order_date,
    TO_CHAR(order_date, 'YYYY-MM-DD') AS iso_format,
    TO_CHAR(order_date, 'Month DD, YYYY') AS full_format,
    TO_CHAR(order_date, 'MM/DD/YY') AS short_format,
    TO_CHAR(order_date, 'YYYY-MM') AS year_month,
    TO_CHAR(order_date, 'Q') AS quarter_display
FROM orders
LIMIT 10;
```

## Complete Date Function Examples

```sql
-- Comprehensive date analysis

-- 1. Customer Tenure Analysis
SELECT 
    customer_id,
    created_date,
    CURRENT_DATE() AS today,
    DATEDIFF(MONTH, created_date, CURRENT_DATE()) AS tenure_months,
    CASE
        WHEN DATEDIFF(MONTH, created_date, CURRENT_DATE()) < 3 THEN 'New'
        WHEN DATEDIFF(MONTH, created_date, CURRENT_DATE()) < 12 THEN 'Growing'
        ELSE 'Established'
    END AS tenure_category
FROM customers
ORDER BY tenure_months DESC;

-- 2. Monthly Trend Analysis
SELECT 
    DATE_TRUNC('MONTH', order_date) AS month,
    COUNT(*) AS order_count,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(order_amount) AS monthly_revenue,
    SUM(SUM(order_amount)) OVER (ORDER BY DATE_TRUNC('MONTH', order_date)) AS cumulative_revenue
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date)
ORDER BY month DESC;

-- 3. Quarterly Performance
SELECT 
    YEAR(order_date) AS year,
    QUARTER(order_date) AS quarter,
    COUNT(*) AS order_count,
    SUM(order_amount) AS quarterly_revenue,
    AVG(order_amount) AS avg_order_value
FROM orders
GROUP BY YEAR(order_date), QUARTER(order_date)
ORDER BY year DESC, quarter DESC;

-- 4. Order Fulfillment Time Analysis
SELECT 
    order_id,
    order_date,
    delivered_date,
    DATEDIFF(DAY, order_date, delivered_date) AS fulfillment_days,
    CASE
        WHEN DATEDIFF(DAY, order_date, delivered_date) <= 3 THEN 'Express'
        WHEN DATEDIFF(DAY, order_date, delivered_date) <= 7 THEN 'Standard'
        ELSE 'Slow'
    END AS fulfillment_type
FROM orders
WHERE delivered_date IS NOT NULL
ORDER BY fulfillment_days DESC;

-- 5. Holiday and Peak Period Identification
SELECT 
    order_date,
    MONTH(order_date) AS month,
    DAY(order_date) AS day,
    DAYOFWEEK(order_date) AS weekday,
    COUNT(*) AS order_count,
    CASE
        WHEN MONTH(order_date) = 12 AND DAY(order_date) BETWEEN 15 AND 31 THEN 'Holiday'
        WHEN MONTH(order_date) = 11 AND DAY(order_date) BETWEEN 20 AND 30 THEN 'Black Friday'
        WHEN MONTH(order_date) = 1 THEN 'New Year'
        WHEN DAYOFWEEK(order_date) IN (6, 7) THEN 'Weekend'
        ELSE 'Regular'
    END AS period_type
FROM orders
GROUP BY order_date, MONTH(order_date), DAY(order_date), DAYOFWEEK(order_date)
ORDER BY order_count DESC
LIMIT 50;
```

## Date Functions Reference

| Function | Purpose | Example |
|----------|---------|---------|
| CURRENT_DATE() | Today's date | SELECT CURRENT_DATE() |
| CURRENT_TIMESTAMP() | Current date and time | SELECT CURRENT_TIMESTAMP() |
| DATEADD | Add interval | DATEADD(DAY, 7, order_date) |
| DATEDIFF | Difference between dates | DATEDIFF(DAY, start, end) |
| DATE_TRUNC | Truncate to interval | DATE_TRUNC('MONTH', date) |
| YEAR/MONTH/DAY | Extract component | YEAR(date) |
| TO_DATE | Parse string to date | TO_DATE('2025-01-15', 'YYYY-MM-DD') |
| TO_CHAR | Format date to string | TO_CHAR(date, 'YYYY-MM-DD') |

## Time Period Buckets

```sql
-- Create standard time buckets for analysis
SELECT 
    order_date,
    CASE
        WHEN QUARTER(order_date) = 1 THEN 'Q1'
        WHEN QUARTER(order_date) = 2 THEN 'Q2'
        WHEN QUARTER(order_date) = 3 THEN 'Q3'
        WHEN QUARTER(order_date) = 4 THEN 'Q4'
    END || '-' || YEAR(order_date) AS year_quarter,
    CASE
        WHEN DAYOFWEEK(order_date) = 1 THEN 'Sunday'
        WHEN DAYOFWEEK(order_date) = 2 THEN 'Monday'
        WHEN DAYOFWEEK(order_date) = 3 THEN 'Tuesday'
        WHEN DAYOFWEEK(order_date) = 4 THEN 'Wednesday'
        WHEN DAYOFWEEK(order_date) = 5 THEN 'Thursday'
        WHEN DAYOFWEEK(order_date) = 6 THEN 'Friday'
        WHEN DAYOFWEEK(order_date) = 7 THEN 'Saturday'
    END AS day_name
FROM orders
LIMIT 10;
```

## Next Steps

1. **Time Series Analysis:** Use window functions with dates
2. **Date Dimension Table:** Create a reference date table
3. **Fiscal Calendar:** Implement custom fiscal periods

## Learning Outcomes

✅ Extract date components  
✅ Calculate date differences  
✅ Add/subtract intervals  
✅ Handle timezones  
✅ Format dates for reporting  
✅ Create time-based aggregations  

## Related Use Cases

- **Use Case 3:** Basic SELECT Queries
- **Use Case 5:** Aggregation and GROUP BY Operations
- **Use Case 11:** Window Functions for Advanced Analytics

---

**Last Updated:** February 18, 2026
