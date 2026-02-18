# Use Case 5: Aggregation and GROUP BY Operations

## Problem Description

Most analytical queries require summarizing data rather than viewing individual records. You need to:

1. Calculate aggregations (sum, count, average, etc.)
2. Group data by dimensions
3. Filter aggregated results
4. Combine multiple aggregation functions
5. Handle NULL values in aggregations

## Business Context

A company wants to:
- Calculate total revenue by month
- Count customers by country
- Find average order value
- Identify top-performing products
- Analyze customer segmentation

## Solution

### Step 1: Basic Aggregation Functions

```sql
-- Create sample orders table
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    product_id INT,
    order_amount DECIMAL(10, 2),
    order_date DATE,
    quantity INT
);

-- Insert sample data
INSERT INTO orders VALUES
(1, 101, 1, 250.00, '2025-01-15', 2),
(2, 102, 2, 500.00, '2025-01-20', 1),
(3, 101, 3, 175.50, '2025-02-01', 3),
(4, 103, 1, 250.00, '2025-02-05', 2),
(5, 102, 4, 600.00, '2025-02-10', 1),
(6, 101, 2, 500.00, '2025-02-15', 1);
```

### Step 2: Basic Aggregation Functions

```sql
-- COUNT: Number of rows
SELECT COUNT(*) AS total_orders
FROM orders;
-- Result: 6

-- COUNT with condition
SELECT COUNT(*) AS orders_over_300
FROM orders
WHERE order_amount > 300;
-- Result: 3

-- SUM: Total of numeric column
SELECT SUM(order_amount) AS total_revenue
FROM orders;
-- Result: 2275.50

-- AVG: Average of numeric column
SELECT AVG(order_amount) AS average_order_value
FROM orders;
-- Result: 379.25

-- MIN: Minimum value
SELECT MIN(order_amount) AS minimum_order
FROM orders;
-- Result: 175.50

-- MAX: Maximum value
SELECT MAX(order_amount) AS maximum_order
FROM orders;
-- Result: 600.00
```

### Step 3: GROUP BY - Basic Grouping

```sql
-- Group by customer and count orders per customer
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(order_amount) AS total_spent,
    AVG(order_amount) AS avg_order_value
FROM orders
GROUP BY customer_id
ORDER BY total_spent DESC;

-- Result:
-- CUSTOMER_ID | ORDER_COUNT | TOTAL_SPENT | AVG_ORDER_VALUE
-- 101         | 3           | 925.50      | 308.50
-- 102         | 2           | 1100.00     | 550.00
-- 103         | 1           | 250.00      | 250.00
```

### Step 4: GROUP BY with Multiple Dimensions

```sql
-- Group by customer and product
SELECT 
    customer_id,
    product_id,
    COUNT(*) AS order_count,
    SUM(order_amount) AS total_spent,
    SUM(quantity) AS total_quantity
FROM orders
GROUP BY customer_id, product_id
ORDER BY customer_id, total_spent DESC;

-- Result:
-- CUSTOMER_ID | PRODUCT_ID | ORDER_COUNT | TOTAL_SPENT | TOTAL_QUANTITY
-- 101         | 1          | 1           | 250.00      | 2
-- 101         | 2          | 1           | 500.00      | 1
-- 101         | 3          | 1           | 175.50      | 3
-- 102         | 2          | 1           | 500.00      | 1
-- 102         | 4          | 1           | 600.00      | 1
-- 103         | 1          | 1           | 250.00      | 2
```

### Step 5: GROUP BY with Date Functions

```sql
-- Group by month to analyze sales trends
SELECT 
    DATE_TRUNC('MONTH', order_date) AS month,
    COUNT(*) AS order_count,
    SUM(order_amount) AS monthly_revenue,
    AVG(order_amount) AS avg_order_value
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date)
ORDER BY month DESC;

-- Result:
-- MONTH      | ORDER_COUNT | MONTHLY_REVENUE | AVG_ORDER_VALUE
-- 2025-02-01 | 4           | 1425.50         | 356.38
-- 2025-01-01 | 2           | 750.00          | 375.00
```

### Step 6: HAVING - Filter Aggregated Results

```sql
-- Find customers with more than 2 orders
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(order_amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 1
ORDER BY order_count DESC;

-- Result:
-- CUSTOMER_ID | ORDER_COUNT | TOTAL_SPENT
-- 101         | 3           | 925.50
-- 102         | 2           | 1100.00

-- Find customers who spent more than $500
SELECT 
    customer_id,
    SUM(order_amount) AS total_spent,
    COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING SUM(order_amount) > 500
ORDER BY total_spent DESC;
```

### Step 7: Combine GROUP BY and JOIN

```sql
-- Add customers table
CREATE TABLE customers (
    customer_id INT,
    name VARCHAR(100),
    country VARCHAR(100)
);

INSERT INTO customers VALUES
(101, 'John Smith', 'USA'),
(102, 'Sarah Johnson', 'Canada'),
(103, 'Michael Brown', 'USA');

-- Group with joins
SELECT 
    c.customer_id,
    c.name,
    c.country,
    COUNT(o.order_id) AS order_count,
    SUM(o.order_amount) AS total_spent,
    AVG(o.order_amount) AS avg_order_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.country
ORDER BY total_spent DESC;
```

### Step 8: Multiple Aggregations with CASE

```sql
-- Categorize orders and count by category
SELECT 
    CASE
        WHEN order_amount < 300 THEN 'Small'
        WHEN order_amount < 600 THEN 'Medium'
        ELSE 'Large'
    END AS order_size,
    COUNT(*) AS order_count,
    SUM(order_amount) AS total_revenue,
    AVG(order_amount) AS avg_amount
FROM orders
GROUP BY order_size
ORDER BY total_revenue DESC;

-- Result:
-- ORDER_SIZE | ORDER_COUNT | TOTAL_REVENUE | AVG_AMOUNT
-- Large      | 1           | 600.00        | 600.00
-- Medium     | 3           | 1225.50       | 408.50
-- Small      | 2           | 250.00        | 125.00
```

## Complete Aggregation Script

```sql
-- Comprehensive aggregation examples

-- 1. Customer Lifetime Value Analysis
SELECT 
    c.customer_id,
    c.name,
    COUNT(o.order_id) AS total_orders,
    MIN(o.order_date) AS first_order_date,
    MAX(o.order_date) AS last_order_date,
    SUM(o.order_amount) AS customer_lifetime_value,
    AVG(o.order_amount) AS avg_order_value,
    MAX(o.order_amount) AS highest_order,
    MIN(o.order_amount) AS lowest_order
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
HAVING COUNT(o.order_id) > 0
ORDER BY customer_lifetime_value DESC;

-- 2. Monthly Sales Report
SELECT 
    DATE_TRUNC('MONTH', order_date) AS sales_month,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(*) AS total_orders,
    SUM(order_amount) AS monthly_revenue,
    AVG(order_amount) AS avg_order,
    MIN(order_amount) AS min_order,
    MAX(order_amount) AS max_order
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date)
ORDER BY sales_month DESC;

-- 3. Product Performance Analysis
SELECT 
    product_id,
    COUNT(*) AS times_ordered,
    SUM(quantity) AS total_units_sold,
    SUM(order_amount) AS total_revenue,
    AVG(order_amount) AS avg_order_value,
    ROUND(SUM(order_amount) / NULLIF(SUM(quantity), 0), 2) AS avg_price_per_unit
FROM orders
GROUP BY product_id
ORDER BY total_revenue DESC;
```

## Advanced Aggregation Patterns

### Pattern 1: Aggregate with DISTINCT

```sql
-- Count unique customers who made purchases
SELECT 
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(*) AS total_orders,
    ROUND(COUNT(*) / COUNT(DISTINCT customer_id), 2) AS orders_per_customer
FROM orders;
```

### Pattern 2: Conditional Aggregation

```sql
-- Use CASE within aggregate functions
SELECT 
    DATE_TRUNC('MONTH', order_date) AS month,
    SUM(CASE WHEN order_amount > 500 THEN order_amount ELSE 0 END) AS large_orders_total,
    SUM(CASE WHEN order_amount <= 500 THEN order_amount ELSE 0 END) AS small_orders_total,
    COUNT(CASE WHEN order_amount > 500 THEN 1 END) AS large_order_count,
    COUNT(CASE WHEN order_amount <= 500 THEN 1 END) AS small_order_count
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date);
```

### Pattern 3: Aggregate with NULL Handling

```sql
-- Use COALESCE to handle nulls
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    COALESCE(SUM(order_amount), 0) AS total_spent,
    COALESCE(AVG(order_amount), 0) AS avg_spent
FROM orders
GROUP BY customer_id;
```

### Pattern 4: Cumulative Aggregation

```sql
-- Running total across months
SELECT 
    DATE_TRUNC('MONTH', order_date) AS month,
    SUM(order_amount) AS monthly_revenue,
    SUM(SUM(order_amount)) OVER (ORDER BY DATE_TRUNC('MONTH', order_date)) AS cumulative_revenue
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date)
ORDER BY month;
```

## Aggregation Functions Reference

| Function | Purpose | Syntax | Example |
|----------|---------|--------|---------|
| COUNT | Count rows | COUNT(*), COUNT(column) | COUNT(*) |
| SUM | Total sum | SUM(column) | SUM(order_amount) |
| AVG | Average value | AVG(column) | AVG(order_amount) |
| MIN | Minimum value | MIN(column) | MIN(order_amount) |
| MAX | Maximum value | MAX(column) | MAX(order_amount) |
| STDDEV | Standard deviation | STDDEV(column) | STDDEV(order_amount) |
| VARIANCE | Variance | VARIANCE(column) | VARIANCE(order_amount) |
| MEDIAN | Median value | MEDIAN(column) | MEDIAN(order_amount) |

## Query Optimization Tips

### 1. Minimize Columns in GROUP BY

```sql
-- Inefficient: Grouping by unnecessary columns
SELECT customer_id, order_id, COUNT(*) FROM orders GROUP BY customer_id, order_id;

-- Efficient: Only necessary columns
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;
```

### 2. Filter Before Grouping with WHERE

```sql
-- Bad: Aggregates everything then filters
SELECT customer_id, SUM(order_amount) FROM orders
GROUP BY customer_id
HAVING SUM(order_amount) > 500;

-- Better: Filter with WHERE first
SELECT customer_id, SUM(order_amount) FROM orders
WHERE order_date >= '2025-01-01'
GROUP BY customer_id
HAVING SUM(order_amount) > 500;
```

### 3. Use Appropriate Data Types

```sql
-- Use DECIMAL for money, not FLOAT
CREATE TABLE orders (
    order_amount DECIMAL(10, 2)  -- Good
    -- order_amount FLOAT        -- Bad for currency
);
```

## Common Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Column must appear in GROUP BY | Non-aggregated column | Add column to GROUP BY or apply aggregate function |
| GROUP BY clause missing | Using aggregate without grouping | Add GROUP BY or remove aggregate |
| HAVING clause invalid | Using column alias from SELECT | Use original column in HAVING |

## Performance Monitoring

```sql
-- Check query execution with aggregations
EXPLAIN
SELECT 
    DATE_TRUNC('MONTH', order_date) AS month,
    SUM(order_amount) AS revenue
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date);
```

## Next Steps

1. **Window Functions:** Advanced analytics with ROW_NUMBER, RANK, etc. (Use Case 11)
2. **Time Series Analysis:** Analyze trends over time
3. **Data Quality:** Validate aggregation results

## Learning Outcomes

✅ Use COUNT, SUM, AVG, MIN, MAX functions  
✅ Group data by multiple dimensions  
✅ Filter aggregations with HAVING  
✅ Combine aggregations with JOINs  
✅ Apply conditional aggregations  
✅ Optimize aggregation queries  

## Related Use Cases

- **Use Case 3:** Basic SELECT Queries
- **Use Case 8:** Joining Multiple Tables
- **Use Case 11:** Window Functions for Advanced Analytics
- **Use Case 25:** Monitoring Queries and Resource Usage

---

**Last Updated:** February 18, 2026
