# Use Case 19: Creating Dynamic Queries with Variables

## Problem Description

Variables enable you to:

1. Create parameterized queries
2. Store intermediate values
3. Reuse values across queries
4. Make queries more maintainable
5. Enable dynamic filtering

## Business Context

Analysts need parameterized queries to:
- Change date ranges easily
- Switch between environments
- Parameterize business rules
- Avoid hardcoding values

## Solution

### Session Variables

```sql
-- Set session variable
SET customer_threshold = 100;
SET min_order_amount = 500;
SET date_start = '2025-01-01'::DATE;

-- Use variables in queries
SELECT 
    customer_id,
    name,
    COUNT(order_id) AS order_count,
    SUM(order_amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE order_amount >= $min_order_amount
  AND order_date >= $date_start
GROUP BY c.customer_id, c.name
HAVING COUNT(order_id) >= $customer_threshold
ORDER BY total_spent DESC;

-- Reference variable with $variable_name
SELECT * FROM customers WHERE customer_id = $customer_id;
```

### Local Variables in Procedures

```sql
-- Declare and use local variables
CREATE OR REPLACE PROCEDURE dynamic_customer_report(
    p_days_back INT DEFAULT 30,
    p_min_amount DECIMAL(10,2) DEFAULT 100
)
RETURNS TABLE(customer_id INT, total_spent DECIMAL(10,2))
LANGUAGE SQL
AS
$$
DECLARE
    v_start_date DATE := DATEADD(DAY, -p_days_back, CURRENT_DATE());
    v_query_date DATE := CURRENT_DATE();
BEGIN
    RETURN TABLE(
        SELECT 
            c.customer_id,
            SUM(o.order_amount) AS total_spent
        FROM customers c
        LEFT JOIN orders o ON c.customer_id = o.customer_id
        WHERE o.order_date >= v_start_date
          AND o.order_date <= v_query_date
          AND o.order_amount >= p_min_amount
        GROUP BY c.customer_id
    );
END;
$$;

-- Execute with different parameters
CALL dynamic_customer_report(30, 100);
CALL dynamic_customer_report(90, 500);
```

### Conditional Variables

```sql
-- Variables with conditional logic
SET environment = 'PRODUCTION';

-- Use in WHERE clause
SELECT * FROM customers
WHERE environment = 'PRODUCTION'
  AND customer_id > 0;

-- Variables in CASE statements
SET discount_level = 'SILVER';

SELECT 
    customer_id,
    name,
    CASE
        WHEN $discount_level = 'PLATINUM' THEN order_amount * 0.20
        WHEN $discount_level = 'GOLD' THEN order_amount * 0.15
        WHEN $discount_level = 'SILVER' THEN order_amount * 0.10
        ELSE order_amount * 0.05
    END AS discounted_amount
FROM orders;
```

## Next Steps

1. **Dynamic SQL:** Build dynamic SQL in procedures
2. **Parameter Validation:** Validate parameter values
3. **Reusable Queries:** Create parameterized query templates

## Learning Outcomes

✅ Create session variables  
✅ Use variables in queries  
✅ Implement dynamic filtering  
✅ Pass parameters to procedures  

---

**Last Updated:** February 18, 2026
