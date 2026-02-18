# Use Case 14: User-Defined Functions (UDFs)

## Problem Description

User-Defined Functions allow you to:

1. Encapsulate business logic for reuse
2. Create custom calculations
3. Simplify complex queries
4. Implement domain-specific operations
5. Improve code maintainability

## Business Context

A company needs:
- Calculate customer lifetime value
- Determine shipping costs based on distance
- Validate business rules
- Transform data consistently
- Hide implementation details

## Solution

### Simple SQL UDF

```sql
-- Basic function returning scalar value
CREATE OR REPLACE FUNCTION calculate_customer_lifetime_value(customer_id INT)
RETURNS DECIMAL(15,2)
LANGUAGE SQL
AS
$$
    SELECT COALESCE(SUM(order_amount), 0)
    FROM orders
    WHERE customer_id = $1
$$;

-- Use the function in queries
SELECT 
    customer_id,
    name,
    calculate_customer_lifetime_value(customer_id) AS lifetime_value
FROM customers
WHERE calculate_customer_lifetime_value(customer_id) > 1000
ORDER BY lifetime_value DESC;
```

### UDF with Multiple Parameters

```sql
-- Function with multiple parameters
CREATE OR REPLACE FUNCTION calculate_discount(
    order_amount DECIMAL(10,2),
    customer_status VARCHAR
)
RETURNS DECIMAL(10,2)
LANGUAGE SQL
AS
$$
    CASE
        WHEN customer_status = 'PLATINUM' THEN order_amount * 0.20
        WHEN customer_status = 'GOLD' THEN order_amount * 0.15
        WHEN customer_status = 'SILVER' THEN order_amount * 0.10
        ELSE order_amount * 0.05
    END
$$;

-- Use in SELECT
SELECT 
    order_id,
    order_amount,
    customer_status,
    calculate_discount(order_amount, customer_status) AS discount,
    order_amount - calculate_discount(order_amount, customer_status) AS net_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

### UDF Returning Table

```sql
-- Function returning result set
CREATE OR REPLACE FUNCTION get_customer_orders(customer_id INT)
RETURNS TABLE(order_id INT, order_amount DECIMAL(10,2), order_date DATE)
LANGUAGE SQL
AS
$$
    SELECT 
        order_id,
        order_amount,
        order_date
    FROM orders
    WHERE customer_id = $1
    ORDER BY order_date DESC
$$;

-- Call table function
SELECT * FROM TABLE(get_customer_orders(101));
```

### Conditional Logic in UDF

```sql
-- Function with conditional logic
CREATE OR REPLACE FUNCTION get_customer_tier(total_spending DECIMAL(15,2))
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
    CASE
        WHEN total_spending < 500 THEN 'Bronze'
        WHEN total_spending < 2000 THEN 'Silver'
        WHEN total_spending < 5000 THEN 'Gold'
        ELSE 'Platinum'
    END
$$;

-- Use in analysis
SELECT 
    customer_id,
    name,
    SUM(order_amount) AS total_spent,
    get_customer_tier(SUM(order_amount)) AS tier
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

## Complete UDF Examples

```sql
-- 1. Complex Calculation Function
CREATE OR REPLACE FUNCTION calculate_shipping_cost(
    base_weight DECIMAL(10,2),
    distance INT,
    is_express BOOLEAN
)
RETURNS DECIMAL(10,2)
LANGUAGE SQL
AS
$$
    CASE
        WHEN is_express THEN
            (base_weight * 0.50 + distance * 0.01) * 1.5
        ELSE
            (base_weight * 0.25 + distance * 0.005)
    END
$$;

-- Test function
SELECT 
    calculate_shipping_cost(5.5, 100, FALSE) AS standard,
    calculate_shipping_cost(5.5, 100, TRUE) AS express;

-- 2. Validation Function
CREATE OR REPLACE FUNCTION is_valid_email(email_address VARCHAR)
RETURNS BOOLEAN
LANGUAGE SQL
AS
$$
    email_address LIKE '%@%.%'
    AND LENGTH(email_address) <= 100
    AND POSITION('@' IN email_address) > 1
    AND POSITION('.' IN email_address) > POSITION('@' IN email_address) + 1
$$;

-- Use in data quality check
SELECT 
    customer_id,
    email,
    is_valid_email(email) AS is_valid
FROM customers
WHERE NOT is_valid_email(email);

-- 3. Aggregation Helper Function
CREATE OR REPLACE FUNCTION calculate_average_order_value_by_month(
    customer_id INT,
    month_count INT
)
RETURNS DECIMAL(10,2)
LANGUAGE SQL
AS
$$
    SELECT COALESCE(AVG(order_amount), 0)
    FROM orders
    WHERE customer_id = $1
      AND order_date >= DATEADD(MONTH, -$2, CURRENT_DATE())
$$;

-- Use in analysis
SELECT 
    customer_id,
    calculate_average_order_value_by_month(customer_id, 3) AS avg_3mo,
    calculate_average_order_value_by_month(customer_id, 6) AS avg_6mo,
    calculate_average_order_value_by_month(customer_id, 12) AS avg_12mo
FROM customers;
```

## Python UDFs

```sql
-- Python UDF for complex logic
CREATE OR REPLACE FUNCTION analyze_text(input_text VARCHAR)
RETURNS OBJECT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
HANDLER = 'analyze_text'
AS
$$
def analyze_text(text):
    return {
        'length': len(text),
        'word_count': len(text.split()),
        'uppercase_count': sum(1 for c in text if c.isupper()),
        'digit_count': sum(1 for c in text if c.isdigit())
    }
$$;

-- Use Python UDF
SELECT 
    customer_id,
    name,
    analyze_text(name) AS name_analysis
FROM customers
LIMIT 5;
```

## UDF Function Reference

| Feature | Description | Example |
|---------|-------------|---------|
| Language | SQL, Python, Java, JavaScript | LANGUAGE SQL |
| Returns | Scalar, Table, OBJECT | RETURNS VARCHAR |
| Parameters | Multiple with types | (param1 INT, param2 VARCHAR) |
| Default | Can have defaults | param1 INT DEFAULT 10 |

## UDF Management

```sql
-- Show all UDFs
SHOW FUNCTIONS;

-- Show specific function
SHOW FUNCTIONS LIKE 'calculate%';

-- Get function definition
SELECT GET_DDL('FUNCTION', 'calculate_customer_lifetime_value(INT)');

-- Drop function
DROP FUNCTION IF EXISTS calculate_customer_lifetime_value(INT);

-- List function calls
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE QUERY_TEXT LIKE '%calculate_customer_lifetime_value%';
```

## Next Steps

1. **Performance Optimization:** Cache UDF results
2. **Advanced Functions:** Implement complex business logic
3. **Secure UDFs:** Implement row-level security with UDFs

## Learning Outcomes

✅ Create scalar and table UDFs  
✅ Implement conditional logic  
✅ Write Python UDFs  
✅ Use UDFs in queries  
✅ Manage function lifecycle  

## Related Use Cases

- **Use Case 13:** Creating and Using Stored Procedures
- **Use Case 45:** Building Custom ML Models with Python UDFs

---

**Last Updated:** February 18, 2026
