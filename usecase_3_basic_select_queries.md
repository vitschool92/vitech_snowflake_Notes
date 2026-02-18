# Use Case 3: Basic SELECT Queries

## Problem Description

Once data is loaded into Snowflake, you need to retrieve and analyze it using SELECT queries. Key challenges include:

1. Writing efficient queries to retrieve specific data
2. Filtering data based on multiple conditions
3. Ordering and limiting results
4. Understanding query execution
5. Optimizing queries for performance

This is the fundamental skill for any data analyst or engineer.

## Business Context

A marketing team needs to:
- Identify customers who joined recently
- Filter customers by specific criteria
- Sort customers by creation date
- Export specific customer information
- Generate basic reports

## Solution

### Basic SELECT Query Structure

```sql
SELECT column1, column2, column3
FROM table_name
WHERE condition1
ORDER BY column1
LIMIT 10;
```

### Use Case 3.1: Simple SELECT All Records

```sql
-- Retrieve all customer records
SELECT * FROM customers;

-- Output format:
-- CUSTOMER_ID | NAME            | EMAIL                    | CREATED_DATE
-- 1           | John Smith      | john.smith@example.com   | 2025-01-15
-- 2           | Sarah Johnson   | sarah.johnson@example.com| 2025-01-20
-- 3           | Michael Brown   | michael.brown@example.com| 2025-02-01
```

### Use Case 3.2: SELECT Specific Columns

```sql
-- Retrieve only name and email columns
SELECT 
    name,
    email
FROM customers;

-- This is more efficient than SELECT * for large tables
-- Only retrieves needed columns
```

### Use Case 3.3: WHERE Clause - Basic Filtering

```sql
-- Find customers created after January 20, 2025
SELECT 
    customer_id,
    name,
    email,
    created_date
FROM customers
WHERE created_date > '2025-01-20';

-- Output:
-- CUSTOMER_ID | NAME            | EMAIL                       | CREATED_DATE
-- 3           | Michael Brown   | michael.brown@example.com   | 2025-02-01
-- 4           | Emily Davis     | emily.davis@example.com     | 2025-02-05
```

### Use Case 3.4: Multiple WHERE Conditions

```sql
-- Find customers created after Jan 20 AND customer_id > 1
SELECT 
    customer_id,
    name,
    created_date
FROM customers
WHERE created_date > '2025-01-20'
  AND customer_id > 1;

-- Using OR condition
SELECT 
    customer_id,
    name
FROM customers
WHERE customer_id = 1 OR customer_id = 3;
```

### Use Case 3.5: String Pattern Matching (LIKE)

```sql
-- Find customers with 'Smith' in their name
SELECT 
    customer_id,
    name,
    email
FROM customers
WHERE name LIKE '%Smith%';

-- Find emails from specific domain
SELECT 
    customer_id,
    name,
    email
FROM customers
WHERE email LIKE '%@example.com%';

-- Pattern matching operators:
-- % - matches any number of characters
-- _ - matches exactly one character
```

### Use Case 3.6: IN Clause - Multiple Values

```sql
-- Find specific customers by ID
SELECT 
    customer_id,
    name,
    email
FROM customers
WHERE customer_id IN (1, 3, 5);

-- Equivalent to:
-- WHERE customer_id = 1 OR customer_id = 3 OR customer_id = 5
```

### Use Case 3.7: BETWEEN - Range Filtering

```sql
-- Find customers created between specific dates
SELECT 
    customer_id,
    name,
    created_date
FROM customers
WHERE created_date BETWEEN '2025-01-15' AND '2025-02-01';

-- Numeric range
SELECT * FROM customers
WHERE customer_id BETWEEN 2 AND 4;
```

### Use Case 3.8: NULL Handling

```sql
-- Find customers with missing email addresses
SELECT 
    customer_id,
    name,
    email
FROM customers
WHERE email IS NULL;

-- Find customers WITH email addresses
SELECT 
    customer_id,
    name,
    email
FROM customers
WHERE email IS NOT NULL;
```

### Use Case 3.9: ORDER BY - Sorting Results

```sql
-- Sort customers by creation date (ascending)
SELECT 
    customer_id,
    name,
    created_date
FROM customers
ORDER BY created_date;

-- Sort by date descending (newest first)
SELECT 
    customer_id,
    name,
    created_date
FROM customers
ORDER BY created_date DESC;

-- Sort by multiple columns
SELECT 
    customer_id,
    name,
    created_date
FROM customers
ORDER BY created_date DESC, name ASC;
```

### Use Case 3.10: LIMIT and OFFSET

```sql
-- Get first 10 customers
SELECT 
    customer_id,
    name
FROM customers
LIMIT 10;

-- Get 10 customers starting from position 20 (pagination)
SELECT 
    customer_id,
    name
FROM customers
ORDER BY customer_id
LIMIT 10 OFFSET 20;

-- Equivalent syntax
SELECT 
    customer_id,
    name
FROM customers
ORDER BY customer_id
LIMIT 10 OFFSET 20;
```

### Use Case 3.11: DISTINCT - Remove Duplicates

```sql
-- Find all unique countries where customers are from
CREATE TABLE customers_enhanced (
    customer_id INT,
    name VARCHAR(100),
    country VARCHAR(100)
);

SELECT DISTINCT country
FROM customers_enhanced
ORDER BY country;

-- Count distinct values
SELECT COUNT(DISTINCT country) AS unique_countries
FROM customers_enhanced;
```

### Use Case 3.12: CASE Statement - Conditional Logic

```sql
-- Categorize customers by registration date
SELECT 
    customer_id,
    name,
    created_date,
    CASE
        WHEN created_date >= '2025-02-01' THEN 'New'
        WHEN created_date >= '2025-01-01' THEN 'Recent'
        ELSE 'Established'
    END AS customer_category
FROM customers
ORDER BY created_date DESC;

-- Output:
-- CUSTOMER_ID | NAME            | CREATED_DATE | CUSTOMER_CATEGORY
-- 5           | David Wilson    | 2025-02-10   | New
-- 4           | Emily Davis     | 2025-02-05   | New
-- 3           | Michael Brown   | 2025-02-01   | New
-- 2           | Sarah Johnson   | 2025-01-20   | Recent
-- 1           | John Smith      | 2025-01-15   | Recent
```

### Use Case 3.13: COALESCE - Handle NULL Values

```sql
-- Replace NULL with default value
SELECT 
    customer_id,
    name,
    COALESCE(email, 'No Email Provided') AS email
FROM customers;

-- Use first non-NULL value from multiple columns
SELECT 
    customer_id,
    COALESCE(alternate_email, primary_email, 'No Email') AS email
FROM customers;
```

### Use Case 3.14: CAST - Type Conversion

```sql
-- Convert data types
SELECT 
    customer_id::VARCHAR AS id_as_string,
    CAST(customer_id AS VARCHAR) AS id_cast_syntax,
    name,
    created_date::VARCHAR AS date_as_string
FROM customers;

-- Common conversions
-- INT::VARCHAR - number to string
-- VARCHAR::INT - string to number
-- DATE::VARCHAR - date to string
-- VARCHAR::DATE - string to date
-- VARCHAR::TIMESTAMP - string to timestamp
```

## Complete Query Examples

### Example 1: Customer Summary Report

```sql
SELECT 
    customer_id,
    name,
    email,
    created_date,
    DATEDIFF(DAY, created_date, CURRENT_DATE()) AS days_as_customer,
    CASE
        WHEN DATEDIFF(DAY, created_date, CURRENT_DATE()) < 30 THEN 'New'
        WHEN DATEDIFF(DAY, created_date, CURRENT_DATE()) < 180 THEN 'Regular'
        ELSE 'Established'
    END AS customer_status
FROM customers
WHERE created_date >= DATEADD(MONTH, -6, CURRENT_DATE())
ORDER BY created_date DESC;
```

### Example 2: High-Value Customers

```sql
-- Assuming orders table exists
SELECT 
    c.customer_id,
    c.name,
    c.email,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(o.order_amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.created_date >= '2025-01-01'
GROUP BY c.customer_id, c.name, c.email
HAVING COUNT(o.order_id) >= 5
ORDER BY total_spent DESC;
```

### Example 3: Customer Segmentation

```sql
SELECT 
    customer_id,
    name,
    created_date,
    CASE
        WHEN customer_id IN (1, 2) THEN 'Segment_A'
        WHEN customer_id IN (3, 4) THEN 'Segment_B'
        ELSE 'Segment_C'
    END AS segment,
    CASE
        WHEN created_date < '2025-01-20' THEN 'Early_Adopter'
        ELSE 'Late_Adopter'
    END AS adoption_type
FROM customers
ORDER BY created_date;
```

## Query Performance Tips

### 1. Avoid SELECT *

```sql
-- Avoid (inefficient for large tables)
SELECT * FROM customers;

-- Prefer (only needed columns)
SELECT customer_id, name, email FROM customers;
```

### 2. Use WHERE Clause to Filter Early

```sql
-- Good (filters rows first)
SELECT customer_id, name
FROM customers
WHERE created_date > '2025-01-01';

-- Avoid (retrieves all then filters)
SELECT customer_id, name
FROM customers
LIMIT 1000000;
```

### 3. Use LIMIT for Large Datasets

```sql
-- Use LIMIT when exploring data
SELECT * FROM customers LIMIT 100;

-- Or use SAMPLE clause
SELECT * FROM customers SAMPLE (10 ROWS);
SELECT * FROM customers SAMPLE (1 PERCENT);
```

## Query Explanation and Optimization

```sql
-- Use EXPLAIN to see query execution plan
EXPLAIN
SELECT 
    customer_id,
    name,
    created_date
FROM customers
WHERE created_date > '2025-01-15'
ORDER BY created_date DESC;

-- Use EXPLAIN PLAN for detailed analysis
EXPLAIN PLAN
SELECT * FROM customers
WHERE customer_id > 10;
```

## Common Query Patterns

### Pattern 1: Find Latest Records

```sql
SELECT * FROM customers
ORDER BY created_date DESC
LIMIT 10;
```

### Pattern 2: Find Specific Value Range

```sql
SELECT * FROM customers
WHERE created_date BETWEEN '2025-01-01' AND '2025-02-28';
```

### Pattern 3: Search Within Text

```sql
SELECT * FROM customers
WHERE name LIKE '%Johnson%'
   OR email LIKE '%gmail.com%';
```

### Pattern 4: Exclude Certain Records

```sql
SELECT * FROM customers
WHERE customer_id NOT IN (1, 2, 3)
  AND email IS NOT NULL;
```

## Comparison Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| = | Equal to | WHERE customer_id = 1 |
| <> or != | Not equal to | WHERE customer_id <> 1 |
| > | Greater than | WHERE created_date > '2025-01-01' |
| < | Less than | WHERE created_date < '2025-02-01' |
| >= | Greater than or equal | WHERE customer_id >= 5 |
| <= | Less than or equal | WHERE customer_id <= 10 |

## Logical Operators

| Operator | Purpose | Example |
|----------|---------|---------|
| AND | Both conditions true | WHERE id > 1 AND name LIKE 'John%' |
| OR | Either condition true | WHERE customer_id = 1 OR customer_id = 2 |
| NOT | Negate condition | WHERE NOT (customer_id = 1) |

## Next Steps

1. **Join Tables:** Learn to combine data from multiple tables (Use Case 8)
2. **Aggregate Data:** Calculate summaries with GROUP BY (Use Case 5)
3. **Window Functions:** Advanced analytical queries (Use Case 11)
4. **Query Optimization:** Improve performance (Use Case 42)

## Learning Outcomes

✅ Write basic SELECT queries  
✅ Filter data with WHERE clauses  
✅ Sort and limit results  
✅ Handle NULL values  
✅ Apply conditional logic  
✅ Optimize query performance  

## Related Use Cases

- **Use Case 1:** Creating Your First Database and Table
- **Use Case 5:** Aggregation and GROUP BY Operations
- **Use Case 8:** Joining Multiple Tables
- **Use Case 11:** Window Functions for Advanced Analytics

---

**Last Updated:** February 18, 2026
