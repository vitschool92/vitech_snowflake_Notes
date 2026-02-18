# Use Case 13: Creating and Using Stored Procedures

## Problem Description

Stored procedures allow you to:

1. Encapsulate complex business logic
2. Execute multiple statements in sequence
3. Implement control flow (IF/THEN, loops)
4. Return results or status
5. Reuse code across applications

## Business Context

A company needs to:
- Update customer status when they place an order
- Calculate monthly commissions
- Archive old data
- Send notifications based on conditions
- Execute multi-step workflows

## Solution

### Basic Stored Procedure

```sql
-- Create a simple stored procedure
CREATE OR REPLACE PROCEDURE update_customer_status(customer_id INT, new_status VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    UPDATE customers 
    SET status = new_status, updated_date = CURRENT_TIMESTAMP()
    WHERE customer_id = $1;
    
    RETURN 'Customer ' || $1 || ' status updated to ' || new_status;
END;
$$;

-- Execute procedure
CALL update_customer_status(101, 'ACTIVE');

-- Result: Customer 101 status updated to ACTIVE
```

### Procedure with Multiple Operations

```sql
-- Procedure with multiple statements
CREATE OR REPLACE PROCEDURE process_monthly_orders(month_year VARCHAR)
RETURNS TABLE(order_count INT, total_revenue DECIMAL(15,2), avg_order_value DECIMAL(10,2))
LANGUAGE SQL
AS
$$
BEGIN
    RETURN TABLE(
        SELECT 
            COUNT(*) AS order_count,
            SUM(order_amount) AS total_revenue,
            AVG(order_amount) AS avg_order_value
        FROM orders
        WHERE TO_CHAR(order_date, 'YYYY-MM') = month_year
    );
END;
$$;

-- Call procedure
CALL process_monthly_orders('2025-02');

-- Result:
-- ORDER_COUNT | TOTAL_REVENUE | AVG_ORDER_VALUE
-- 4           | 1425.50       | 356.38
```

### Procedure with Control Flow

```sql
-- Procedure with IF/THEN/ELSE
CREATE OR REPLACE PROCEDURE categorize_customer(customer_id INT)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    total_spent DECIMAL(15,2);
    category VARCHAR;
BEGIN
    -- Calculate customer spending
    SELECT SUM(order_amount) INTO total_spent
    FROM orders
    WHERE customer_id = $1;
    
    -- Categorize based on spending
    IF total_spent IS NULL THEN
        category := 'NEW';
    ELSIF total_spent < 500 THEN
        category := 'BASIC';
    ELSIF total_spent < 2000 THEN
        category := 'SILVER';
    ELSIF total_spent < 5000 THEN
        category := 'GOLD';
    ELSE
        category := 'PLATINUM';
    END IF;
    
    -- Update customer category
    UPDATE customers 
    SET customer_category = category
    WHERE customer_id = $1;
    
    RETURN 'Customer ' || $1 || ' categorized as ' || category;
END;
$$;

-- Execute procedure
CALL categorize_customer(101);
```

### Procedure with Loop

```sql
-- Procedure with loop
CREATE OR REPLACE PROCEDURE process_multiple_customers(customer_ids ARRAY)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    processed_count INT := 0;
    current_id INT;
    i INT := 0;
BEGIN
    WHILE i < ARRAY_LENGTH($1) DO
        current_id := $1[i + 1];
        
        -- Process each customer
        UPDATE customers 
        SET processed_flag = TRUE,
            processed_date = CURRENT_TIMESTAMP()
        WHERE customer_id = current_id;
        
        processed_count := processed_count + 1;
        i := i + 1;
    END WHILE;
    
    RETURN 'Processed ' || processed_count || ' customers';
END;
$$;

-- Execute procedure
CALL process_multiple_customers(ARRAY[101, 102, 103]);
```

## Complete Procedure Examples

```sql
-- 1. Customer Lifecycle Procedure
CREATE OR REPLACE PROCEDURE customer_lifecycle_update(
    p_customer_id INT, 
    p_last_purchase_days INT DEFAULT 30
)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    v_days_since_purchase INT;
    v_new_status VARCHAR;
    v_message VARCHAR;
BEGIN
    -- Get days since last purchase
    SELECT DATEDIFF(DAY, MAX(order_date), CURRENT_DATE())
    INTO v_days_since_purchase
    FROM orders
    WHERE customer_id = p_customer_id;
    
    -- Determine status
    IF v_days_since_purchase IS NULL THEN
        v_new_status := 'NO_PURCHASES';
    ELSIF v_days_since_purchase <= 30 THEN
        v_new_status := 'ACTIVE';
    ELSIF v_days_since_purchase <= 90 THEN
        v_new_status := 'AT_RISK';
    ELSE
        v_new_status := 'DORMANT';
    END IF;
    
    -- Update customer
    UPDATE customers
    SET status = v_new_status, updated_date = CURRENT_TIMESTAMP()
    WHERE customer_id = p_customer_id;
    
    v_message := 'Customer ' || p_customer_id || ' status: ' || v_new_status;
    RETURN v_message;
END;
$$;

-- Execute procedure
CALL customer_lifecycle_update(101, 30);

-- 2. Data Archive Procedure
CREATE OR REPLACE PROCEDURE archive_old_orders(archive_months INT DEFAULT 12)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    v_archived_count INT;
    v_archive_date DATE;
BEGIN
    v_archive_date := DATEADD(MONTH, -archive_months, CURRENT_DATE());
    
    -- Move to archive table
    INSERT INTO orders_archive
    SELECT * FROM orders
    WHERE order_date < v_archive_date;
    
    GET DIAGNOSTICS v_archived_count = ROW_COUNT;
    
    -- Delete from main table
    DELETE FROM orders
    WHERE order_date < v_archive_date;
    
    RETURN 'Archived ' || v_archived_count || ' orders older than ' || v_archive_date;
END;
$$;

-- Execute procedure
CALL archive_old_orders(12);
```

## Procedure Parameter Handling

```sql
-- Parameters with defaults
CREATE OR REPLACE PROCEDURE calculate_discount(
    customer_id INT,
    discount_percent DECIMAL(5,2) DEFAULT 10,
    minimum_order DECIMAL(10,2) DEFAULT 100
)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    UPDATE orders
    SET discount = order_amount * (discount_percent / 100)
    WHERE customer_id = $1
      AND order_amount >= minimum_order;
    
    RETURN 'Discount applied';
END;
$$;

-- Call with different parameter combinations
CALL calculate_discount(101);  -- Uses defaults
CALL calculate_discount(101, 15);  -- Custom discount
CALL calculate_discount(101, 15, 500);  -- All custom
```

## Error Handling in Procedures

```sql
-- Procedure with error handling
CREATE OR REPLACE PROCEDURE safe_update_customer(
    customer_id INT,
    new_email VARCHAR
)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    -- Try to update
    UPDATE customers
    SET email = new_email
    WHERE customer_id = customer_id;
    
    IF ROW_COUNT() = 0 THEN
        RAISE EXCEPTION 'Customer not found';
    END IF;
    
    RETURN 'Successfully updated customer';
EXCEPTION
    WHEN OTHER THEN
        RETURN 'Error: ' || SQLERRM;
END;
$$;

-- Execute procedure
CALL safe_update_customer(101, 'newemail@example.com');
```

## Procedure Management

```sql
-- Show all procedures
SHOW PROCEDURES;

-- Show procedure details
SHOW PROCEDURES LIKE 'update_customer%';

-- Get procedure definition
SELECT GET_DDL('PROCEDURE', 'update_customer_status(INT, VARCHAR)');

-- Drop procedure
DROP PROCEDURE IF EXISTS update_customer_status(INT, VARCHAR);
```

## Next Steps

1. **Automate with Tasks:** Schedule procedures to run automatically
2. **Error Handling:** Implement comprehensive error management
3. **Procedure Optimization:** Performance tune complex procedures

## Learning Outcomes

✅ Create stored procedures  
✅ Implement control flow logic  
✅ Handle parameters  
✅ Return results  
✅ Implement error handling  

## Related Use Cases

- **Use Case 14:** User-Defined Functions (UDFs)
- **Use Case 31:** Building ETL Pipelines with Tasks and Streams

---

**Last Updated:** February 18, 2026
