# Use Case 8: Joining Multiple Tables

## Problem Description

Real-world data is normalized across multiple tables. You need to:

1. Understand different join types
2. Combine data from multiple tables
3. Handle complex join conditions
4. Optimize join performance
5. Debug join issues

## Business Context

Customer and order data are in separate tables:
- Customers table: customer info
- Orders table: order details linked to customers
- Products table: product information
- Need to analyze customer behavior with their orders and products

## Solution

### Types of Joins

```sql
-- Create sample tables
CREATE TABLE customers (
    customer_id INT,
    name VARCHAR(100),
    country VARCHAR(50)
);

CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    order_amount DECIMAL(10, 2),
    order_date DATE
);

CREATE TABLE order_items (
    item_id INT,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10, 2)
);

-- Insert sample data
INSERT INTO customers VALUES 
(1, 'John Smith', 'USA'),
(2, 'Sarah Johnson', 'Canada'),
(3, 'Michael Brown', 'USA');

INSERT INTO orders VALUES 
(101, 1, 250.00, '2025-01-15'),
(102, 2, 500.00, '2025-01-20'),
(103, 1, 175.50, '2025-02-01');

INSERT INTO order_items VALUES 
(1, 101, 1, 2, 125.00),
(2, 101, 2, 1, 125.00),
(3, 102, 3, 1, 500.00);
```

### INNER JOIN - Only Matching Records

```sql
-- Inner join: returns only rows with matches in both tables
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.order_amount,
    o.order_date
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- Result: Only customers with orders (3 rows)
-- CUSTOMER_ID | NAME           | ORDER_ID | ORDER_AMOUNT | ORDER_DATE
-- 1           | John Smith     | 101      | 250.00       | 2025-01-15
-- 2           | Sarah Johnson  | 102      | 500.00       | 2025-01-20
-- 1           | John Smith     | 103      | 175.50       | 2025-02-01
```

### LEFT JOIN - All from Left Table

```sql
-- Left join: all rows from left table, matching from right
SELECT 
    c.customer_id,
    c.name,
    COUNT(o.order_id) AS order_count,
    COALESCE(SUM(o.order_amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;

-- Result: All customers, with order counts
-- CUSTOMER_ID | NAME           | ORDER_COUNT | TOTAL_SPENT
-- 1           | John Smith     | 2           | 425.50
-- 2           | Sarah Johnson  | 1           | 500.00
-- 3           | Michael Brown  | 0           | 0.00
```

### RIGHT JOIN - All from Right Table

```sql
-- Right join: all rows from right table, matching from left
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.order_amount
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;

-- Equivalent to LEFT JOIN with tables reversed
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.order_amount
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

### FULL OUTER JOIN - All Records

```sql
-- Full outer join: all rows from both tables
SELECT 
    COALESCE(c.customer_id, o.customer_id) AS customer_id,
    c.name,
    o.order_id,
    o.order_amount
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
ORDER BY customer_id, order_id;
```

### CROSS JOIN - Cartesian Product

```sql
-- Cross join: combines every row from left with every row from right
SELECT 
    c.customer_id,
    c.name,
    o.order_id
FROM customers c
CROSS JOIN orders o
LIMIT 10;

-- Result: 3 customers × 3 orders = 9 rows
-- Useful for generating combinations
```

### Multiple Joins - Joining 3+ Tables

```sql
-- Join customers, orders, and order_items
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.order_amount,
    oi.product_id,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS line_total
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
ORDER BY c.customer_id, o.order_id;
```

## Complete Join Examples

### Example 1: Customer Order History with Details

```sql
SELECT 
    c.customer_id,
    c.name,
    c.country,
    o.order_id,
    o.order_date,
    o.order_amount,
    COUNT(oi.item_id) AS items_in_order,
    SUM(oi.quantity) AS total_quantity
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name, c.country, o.order_id, o.order_date, o.order_amount
ORDER BY c.customer_id, o.order_date DESC;
```

### Example 2: Find Customers Without Orders

```sql
-- Identify inactive customers
SELECT 
    c.customer_id,
    c.name,
    c.country
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL
ORDER BY c.customer_id;
```

### Example 3: Join with Aggregation

```sql
-- Customer lifetime value with detailed breakdown
SELECT 
    c.customer_id,
    c.name,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT oi.product_id) AS unique_products,
    SUM(oi.quantity) AS total_items_purchased,
    SUM(oi.quantity * oi.unit_price) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name
HAVING COUNT(DISTINCT o.order_id) > 0
ORDER BY lifetime_value DESC;
```

### Example 4: Self-Join - Compare Data in Same Table

```sql
-- Find customers from same country
CREATE TABLE customer_connections AS
SELECT 
    c1.customer_id,
    c1.name,
    c2.customer_id AS peer_customer_id,
    c2.name AS peer_name,
    c1.country
FROM customers c1
INNER JOIN customers c2 
    ON c1.country = c2.country 
    AND c1.customer_id < c2.customer_id;

-- Result: Groups customers by country
```

## Join Conditions and Filtering

### Multiple Join Conditions

```sql
-- Join with multiple conditions
SELECT * FROM orders o
INNER JOIN customers c 
    ON o.customer_id = c.customer_id
    AND o.order_date >= '2025-01-01'
    AND c.country = 'USA';
```

### Join with Expressions

```sql
-- Join using expressions
SELECT 
    c.customer_id,
    o.order_id
FROM customers c
INNER JOIN orders o 
    ON c.customer_id = o.customer_id 
    AND YEAR(o.order_date) = 2025;
```

### WHERE vs ON Clause

```sql
-- WRONG: Using WHERE for join reduces to INNER JOIN effect
SELECT * FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NOT NULL;  -- Converts to INNER JOIN!

-- CORRECT: Use ON for join conditions
SELECT * FROM customers c
LEFT JOIN orders o 
    ON c.customer_id = o.customer_id 
    AND o.order_date >= '2025-01-01';

-- Use WHERE to filter after join
SELECT * FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.country = 'USA';
```

## Join Performance Tips

### 1. Join on Indexed Columns

```sql
-- Create indexes on foreign keys
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Joins on indexed columns are faster
SELECT * FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;
```

### 2. Minimize Data Before Join

```sql
-- Bad: Join everything then filter
SELECT * FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2025-01-01';

-- Better: Filter first using subqueries
SELECT * FROM customers c
INNER JOIN (
    SELECT * FROM orders WHERE order_date >= '2025-01-01'
) o ON c.customer_id = o.customer_id;
```

### 3. Use EQUI-JOINS When Possible

```sql
-- Good: Equi-join (equality condition)
SELECT * FROM orders o
INNER JOIN order_items oi ON o.order_id = oi.order_id;

-- Avoid: Non-equi join (slow)
SELECT * FROM orders o
INNER JOIN order_items oi ON o.order_amount > oi.unit_price;
```

## Common Join Patterns

### Pattern 1: Existence Check

```sql
-- Check if customers have orders
SELECT 
    c.customer_id,
    c.name,
    CASE 
        WHEN o.order_id IS NOT NULL THEN 'Active'
        ELSE 'Inactive'
    END AS status
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

### Pattern 2: Master-Detail Pattern

```sql
-- Retrieve customer with all their order details
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.order_date,
    oi.product_id,
    oi.quantity,
    oi.unit_price
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
ORDER BY c.customer_id, o.order_id, oi.item_id;
```

### Pattern 3: Deduplication with Window Function

```sql
-- Get latest order for each customer
WITH ranked_orders AS (
    SELECT 
        c.customer_id,
        c.name,
        o.order_id,
        o.order_date,
        ROW_NUMBER() OVER (PARTITION BY c.customer_id ORDER BY o.order_date DESC) AS rn
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
)
SELECT * FROM ranked_orders WHERE rn = 1;
```

## Troubleshooting Joins

| Issue | Cause | Solution |
|-------|-------|----------|
| Too many rows | Unintended Cartesian product | Add proper join condition |
| Missing rows | Wrong join type | Use LEFT/RIGHT/FULL OUTER as needed |
| Duplicates | Joining on non-unique columns | Join on primary/foreign keys |
| Performance slow | Joining on non-indexed columns | Create indexes or filter before join |

## Next Steps

1. **Window Functions:** Advanced analytics with joins
2. **Subqueries:** Alternative to joins for complex queries
3. **Query Optimization:** Profile and tune join performance

## Learning Outcomes

✅ Understand INNER, LEFT, RIGHT, FULL OUTER joins  
✅ Combine multiple tables  
✅ Handle complex join conditions  
✅ Filter joined data correctly  
✅ Optimize join performance  

## Related Use Cases

- **Use Case 3:** Basic SELECT Queries
- **Use Case 5:** Aggregation and GROUP BY Operations
- **Use Case 11:** Window Functions for Advanced Analytics

---

**Last Updated:** February 18, 2026
