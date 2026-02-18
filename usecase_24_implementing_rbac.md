# Use Case 24: Implementing Role-Based Access Control (RBAC)

## Problem Description

Advanced RBAC involves:

1. Column-level masking
2. Row-level access policies
3. Dynamic masking policies
4. Fine-grained permissions

## Business Context

A company needs to:
- Mask sensitive columns for certain roles
- Restrict rows by department
- Enforce data residency

## Solution

```sql
-- Create masking policy
CREATE MASKING POLICY email_mask AS (email VARCHAR)
RETURNS VARCHAR ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'DATA_ENGINEER') THEN email
        ELSE '***@***.***'
    END;

-- Apply policy
ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY email_mask;

-- Create row-level policy
CREATE ROW ACCESS POLICY department_policy AS (dept VARCHAR)
RETURNS BOOLEAN ->
    CASE
        WHEN CURRENT_ROLE() = 'ADMIN' THEN TRUE
        WHEN CURRENT_ROLE() LIKE '%MANAGER%' THEN dept = CURRENT_USER()
        ELSE FALSE
    END;

-- Apply row policy
ALTER TABLE employees ADD ROW ACCESS POLICY department_policy ON (department);

-- Query returns masked data
SELECT customer_id, name, email FROM customers;
-- ADMIN sees real email
-- Others see ***@***.***
```

## Next Steps

1. **Policy Management:** Version and manage policies
2. **Compliance:** Use for GDPR/HIPAA compliance
3. **Audit:** Track policy violations

## Learning Outcomes

✅ Implement masking policies  
✅ Apply row access policies  
✅ Test policy enforcement  

---

**Last Updated:** February 18, 2026
