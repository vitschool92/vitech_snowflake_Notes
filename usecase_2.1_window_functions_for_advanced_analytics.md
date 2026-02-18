# Use Case 11: Window Functions for Advanced Analytics

## Problem Description

Standard GROUP BY aggregations lose row-level details. Window functions allow you to:

1. Calculate running totals
2. Rank rows within groups
3. Access previous/next rows
4. Calculate differences over time
5. Perform row-level aggregations without collapsing rows

## Business Context

A company wants to:
- Show running sales totals
- Rank customers by purchase amount
- Identify top 5 customers per region
- Calculate month-over-month changes
- Find consecutive purchases

## Solution

### ROW_NUMBER() - Unique Ranking

```sql
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    order_amount DECIMAL(10, 2),
    order_date DATE
);

-- ROW_NUMBER: Unique sequential number within group
SELECT 
    customer_id,
    order_id,
    order_amount,
    order_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_sequence,
    ROW_NUMBER() OVER (ORDER BY order_amount DESC) AS amount_rank
FROM orders
ORDER BY customer_id, order_date;

-- Result:
-- CUSTOMER_ID | ORDER_ID | ORDER_AMOUNT | ORDER_DATE | ORDER_SEQUENCE | AMOUNT_RANK
-- 101         | 1001     | 250.00       | 2025-01-15 | 1              | 5
-- 101         | 1003     | 175.50       | 2025-02-01 | 2              | 8
-- 102         | 1002     | 500.00       | 2025-01-20 | 1              | 2
```

### RANK() and DENSE_RANK() - Handling Ties

```sql
-- RANK: Allows gaps when there are ties
-- DENSE_RANK: No gaps, consecutive numbering

SELECT 
    customer_id,
    order_amount,
    RANK() OVER (ORDER BY order_amount DESC) AS rank_with_gaps,
    DENSE_RANK() OVER (ORDER BY order_amount DESC) AS rank_dense
FROM orders
ORDER BY order_amount DESC;

-- Example with ties:
-- $500 → rank 1, dense_rank 1
-- $500 → rank 1, dense_rank 1
-- $250 → rank 3, dense_rank 2
-- $200 → rank 4, dense_rank 3
```

### Top N Records per Group

```sql
-- Find top 3 orders per customer
WITH ranked_orders AS (
    SELECT 
        customer_id,
        order_id,
        order_amount,
        order_date,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_amount DESC) AS rn
    FROM orders
)
SELECT * FROM ranked_orders WHERE rn <= 3
ORDER BY customer_id, order_amount DESC;
```

### Running Total (Cumulative Sum)

```sql
-- Calculate running total of orders
SELECT 
    customer_id,
    order_date,
    order_amount,
    SUM(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    SUM(order_amount) OVER (PARTITION BY customer_id) AS total_customer_spending
FROM orders
ORDER BY customer_id, order_date;

-- Result:
-- CUSTOMER_ID | ORDER_DATE | ORDER_AMOUNT | RUNNING_TOTAL | TOTAL_CUSTOMER_SPENDING
-- 101         | 2025-01-15 | 250.00       | 250.00        | 425.50
-- 101         | 2025-02-01 | 175.50       | 425.50        | 425.50
```

### LAG() and LEAD() - Access Previous/Next Rows

```sql
-- Access previous order amount
SELECT 
    customer_id,
    order_date,
    order_amount,
    LAG(order_amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_amount,
    LEAD(order_amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_amount,
    order_amount - LAG(order_amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS change_from_previous
FROM orders
ORDER BY customer_id, order_date;

-- Result:
-- CUSTOMER_ID | ORDER_DATE | ORDER_AMOUNT | PREV_ORDER_AMOUNT | NEXT_ORDER_AMOUNT | CHANGE_FROM_PREVIOUS
-- 101         | 2025-01-15 | 250.00       | NULL              | 175.50            | NULL
-- 101         | 2025-02-01 | 175.50       | 250.00            | NULL              | -74.50
```

### Moving Average

```sql
-- Calculate 3-order moving average
SELECT 
    customer_id,
    order_date,
    order_amount,
    AVG(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3_orders
FROM orders
ORDER BY customer_id, order_date;
```

### FIRST_VALUE() and LAST_VALUE()

```sql
-- Get first and last order details
SELECT 
    customer_id,
    order_date,
    order_amount,
    FIRST_VALUE(order_amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS first_order_amount,
    LAST_VALUE(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_amount,
    DATEDIFF(DAY, 
        FIRST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date),
        LAST_VALUE(order_date) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        )
    ) AS customer_lifespan_days
FROM orders
ORDER BY customer_id, order_date;
```

## Complete Window Function Examples

```sql
-- 1. Customer Lifetime Value with Trend
SELECT 
    customer_id,
    order_date,
    order_amount,
    SUM(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) AS cumulative_ltv,
    AVG(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3_orders,
    order_amount - LAG(order_amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS order_change
FROM orders
ORDER BY customer_id, order_date;

-- 2. Customer Segmentation by Rank
SELECT 
    customer_id,
    total_spending,
    RANK() OVER (ORDER BY total_spending DESC) AS spending_rank,
    CASE
        WHEN RANK() OVER (ORDER BY total_spending DESC) <= 10 THEN 'Top 10'
        WHEN RANK() OVER (ORDER BY total_spending DESC) <= 100 THEN 'Top 100'
        ELSE 'Regular'
    END AS segment
FROM (
    SELECT 
        customer_id,
        SUM(order_amount) AS total_spending
    FROM orders
    GROUP BY customer_id
);

-- 3. Monthly Growth Analysis
SELECT 
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    SUM(order_amount) AS monthly_revenue,
    LAG(SUM(order_amount)) OVER (ORDER BY YEAR(order_date), MONTH(order_date)) AS prev_month_revenue,
    SUM(order_amount) - LAG(SUM(order_amount)) OVER (ORDER BY YEAR(order_date), MONTH(order_date)) AS revenue_change,
    ROUND(100 * (SUM(order_amount) - LAG(SUM(order_amount)) OVER (ORDER BY YEAR(order_date), MONTH(order_date))) 
          / LAG(SUM(order_amount)) OVER (ORDER BY YEAR(order_date), MONTH(order_date)), 2) AS growth_percent
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY year, month;
```

## Window Frame Specification

```sql
-- Different window frame options

SELECT 
    customer_id,
    order_amount,
    -- Unbounded: from beginning/end of partition
    SUM(order_amount) OVER (PARTITION BY customer_id) AS total,
    
    -- Current row only
    SUM(order_amount) OVER (PARTITION BY customer_id ROWS BETWEEN CURRENT ROW AND CURRENT ROW) AS current_amount,
    
    -- N rows before
    SUM(order_amount) OVER (PARTITION BY customer_id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS sum_3_rows,
    
    -- N rows after
    SUM(order_amount) OVER (PARTITION BY customer_id ROWS BETWEEN CURRENT ROW AND 1 FOLLOWING) AS current_and_next,
    
    -- Unbounded both directions
    SUM(order_amount) OVER (PARTITION BY customer_id ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS grand_total
FROM orders;
```

## Window Function Reference

| Function | Purpose | Example |
|----------|---------|---------|
| ROW_NUMBER() | Sequential number | ROW_NUMBER() OVER (ORDER BY amount) |
| RANK() | Rank with gaps | RANK() OVER (ORDER BY amount DESC) |
| DENSE_RANK() | Rank without gaps | DENSE_RANK() OVER (ORDER BY amount) |
| LAG() | Previous row value | LAG(amount, 1) OVER (ORDER BY date) |
| LEAD() | Next row value | LEAD(amount, 1) OVER (ORDER BY date) |
| SUM() | Running/total sum | SUM(amount) OVER (ORDER BY date) |
| AVG() | Average in window | AVG(amount) OVER (ORDER BY date) |
| FIRST_VALUE() | First value in window | FIRST_VALUE(amount) OVER (ORDER BY date) |
| LAST_VALUE() | Last value in window | LAST_VALUE(amount) OVER (ORDER BY date) |
| PERCENT_RANK() | Percentile rank | PERCENT_RANK() OVER (ORDER BY amount) |
| NTILE() | Divide into buckets | NTILE(4) OVER (ORDER BY amount) |

## Performance Considerations

```sql
-- GOOD: Window function with specific partition
SELECT 
    customer_id,
    order_amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS rn
FROM orders;

-- POTENTIALLY SLOW: Window function across entire table
SELECT 
    customer_id,
    order_amount,
    ROW_NUMBER() OVER (ORDER BY order_amount DESC) AS global_rank
FROM orders WHERE order_amount > 1000;
```

## Next Steps

1. **Advanced Analytics:** Combine with other functions for complex analysis
2. **Performance Tuning:** Optimize window function queries
3. **Real-Time Analysis:** Stream processing with window functions

## Learning Outcomes

✅ Use ROW_NUMBER, RANK, DENSE_RANK  
✅ Calculate running totals  
✅ Access previous/next rows  
✅ Perform group-level calculations  
✅ Handle window frame specifications  

## Related Use Cases

- **Use Case 3:** Basic SELECT Queries
- **Use Case 5:** Aggregation and GROUP BY Operations
- **Use Case 42:** Advanced Query Optimization

---

**Last Updated:** February 18, 2026
