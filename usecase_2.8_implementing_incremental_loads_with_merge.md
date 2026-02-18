# Use Case 18: Implementing Incremental Loads with MERGE

## Problem Description

MERGE statement enables you to:

1. Update matching records
2. Insert new records
3. Delete unmatched records (with WHEN NOT MATCHED BY SOURCE)
4. Handle all scenarios in single operation
5. Implement efficient upserts

## Business Context

Daily customer updates need to:
- Insert new customers
- Update existing customer info
- Handle deletions
- All in one operation for consistency

## Solution

### Basic MERGE Pattern

```sql
-- MERGE: INSERT + UPDATE in single operation
MERGE INTO customers c
USING staging_customers s
ON c.customer_id = s.customer_id
WHEN MATCHED AND c.email != s.email 
    THEN UPDATE SET c.email = s.email, c.updated_date = CURRENT_DATE()
WHEN NOT MATCHED 
    THEN INSERT (customer_id, name, email, created_date)
         VALUES (s.customer_id, s.name, s.email, CURRENT_DATE());

-- Result:
-- - Updates existing customers with new email
-- - Inserts new customers from staging table
```

### MERGE with Conditions

```sql
-- MERGE with WHERE conditions
MERGE INTO orders o
USING order_staging os
ON o.order_id = os.order_id
WHEN MATCHED AND os.status = 'CANCELLED'
    THEN DELETE
WHEN MATCHED AND os.order_amount != o.order_amount
    THEN UPDATE SET o.order_amount = os.order_amount, o.updated_date = CURRENT_TIMESTAMP()
WHEN NOT MATCHED AND os.status != 'CANCELLED'
    THEN INSERT (order_id, customer_id, order_amount, status, created_date)
         VALUES (os.order_id, os.customer_id, os.order_amount, os.status, CURRENT_DATE());
```

### MERGE with Multiple Conditions

```sql
-- Complex MERGE logic
MERGE INTO products p
USING product_staging ps
ON p.product_id = ps.product_id
WHEN MATCHED AND ps.status = 'ACTIVE' AND ps.price != p.price
    THEN UPDATE SET 
        p.price = ps.price,
        p.last_price = p.price,
        p.price_update_date = CURRENT_TIMESTAMP()
WHEN MATCHED AND ps.status = 'DISCONTINUED'
    THEN UPDATE SET
        p.status = 'DISCONTINUED',
        p.discontinued_date = CURRENT_TIMESTAMP()
WHEN NOT MATCHED BY TARGET AND ps.status = 'ACTIVE'
    THEN INSERT (product_id, name, price, status, created_date)
         VALUES (ps.product_id, ps.name, ps.price, ps.status, CURRENT_DATE())
WHEN NOT MATCHED BY SOURCE AND p.status = 'ACTIVE'
    THEN DELETE;
```

## Complete Incremental Load Example

```sql
-- Daily customer update process

-- 1. Load new data to staging
CREATE TEMPORARY TABLE customer_staging (
    customer_id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    updated_date DATE
);

-- Insert staging data (from source system)
INSERT INTO customer_staging VALUES
(1, 'John Smith', 'john.newemail@example.com', '2026-02-18'),
(2, 'Sarah Johnson', 'sarah@example.com', '2026-02-18'),
(4, 'Michael Brown', 'michael@example.com', '2026-02-18');  -- New customer

-- 2. Execute MERGE
MERGE INTO customers c
USING customer_staging cs
ON c.customer_id = cs.customer_id
WHEN MATCHED THEN UPDATE SET
    c.name = cs.name,
    c.email = cs.email,
    c.updated_date = cs.updated_date
WHEN NOT MATCHED THEN INSERT
    (customer_id, name, email, created_date, updated_date)
VALUES
    (cs.customer_id, cs.name, cs.email, CURRENT_DATE(), cs.updated_date);

-- 3. Log results
INSERT INTO merge_audit_log
(table_name, rows_merged, merge_timestamp)
VALUES
('customers', ROW_COUNT(), CURRENT_TIMESTAMP());
```

## MERGE with Subqueries

```sql
-- MERGE with calculated values
MERGE INTO customer_metrics cm
USING (
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(order_amount) AS total_spent
    FROM orders
    GROUP BY customer_id
) o
ON cm.customer_id = o.customer_id
WHEN MATCHED THEN UPDATE SET
    cm.order_count = o.order_count,
    cm.total_spent = o.total_spent,
    cm.last_updated = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT
    (customer_id, order_count, total_spent, last_updated)
VALUES
    (o.customer_id, o.order_count, o.total_spent, CURRENT_TIMESTAMP());
```

## Next Steps

1. **Automate with Tasks:** Schedule MERGE operations
2. **Error Handling:** Add validation before MERGE
3. **Audit Trail:** Track MERGE operations

## Learning Outcomes

✅ Use MERGE for insert/update  
✅ Handle multiple conditions  
✅ Implement efficient upserts  
✅ Delete unmatched records  

---

**Last Updated:** February 18, 2026
