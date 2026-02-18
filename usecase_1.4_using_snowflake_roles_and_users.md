# Use Case 4: Using Snowflake Roles and Users

## Problem Description

Snowflake is a multi-tenant system that requires robust access control. You need to:

1. Create users for different team members
2. Define roles with specific permissions
3. Grant permissions to roles
4. Assign roles to users
5. Implement least-privilege access principle
6. Audit user activities

## Business Context

An organization has different teams with varying data access needs:
- **Data Engineers:** Need CREATE/ALTER permissions on databases
- **Analysts:** Need SELECT permissions on analytics views
- **Executives:** Need SELECT permissions on specific dashboards
- **Administrators:** Full access to all objects

## Solution

### Step 1: Create Roles

```sql
-- Create roles for different job functions
CREATE ROLE admin_role;
CREATE ROLE analyst_role;
CREATE ROLE engineer_role;
CREATE ROLE viewer_role;

-- Verify roles
SHOW ROLES;
```

**Role Hierarchy:**
```
admin_role (Full Access)
├── engineer_role (Create/Modify Objects)
├── analyst_role (Query Data)
└── viewer_role (View Only)
```

### Step 2: Create Users

```sql
-- Create admin user
CREATE USER admin_user
  PASSWORD = 'SecureAdminPass123!'
  EMAIL = 'admin@company.com'
  FIRST_NAME = 'John'
  LAST_NAME = 'Admin'
  DEFAULT_WAREHOUSE = 'compute_wh'
  DEFAULT_ROLE = 'admin_role'
  DEFAULT_NAMESPACE = 'customer_db.public'
  COMMENT = 'System administrator';

-- Create analyst user
CREATE USER analyst_user
  PASSWORD = 'SecureAnalystPass456!'
  EMAIL = 'analyst@company.com'
  FIRST_NAME = 'Sarah'
  LAST_NAME = 'Analyst'
  DEFAULT_WAREHOUSE = 'analytics_wh'
  DEFAULT_ROLE = 'analyst_role';

-- Create data engineer user
CREATE USER engineer_user
  PASSWORD = 'SecureEngineerPass789!'
  EMAIL = 'engineer@company.com'
  DEFAULT_WAREHOUSE = 'compute_wh'
  DEFAULT_ROLE = 'engineer_role';

-- Create viewer user
CREATE USER viewer_user
  PASSWORD = 'SecureViewerPass012!'
  EMAIL = 'viewer@company.com'
  DEFAULT_WAREHOUSE = 'query_wh'
  DEFAULT_ROLE = 'viewer_role';

-- Verify users
SHOW USERS;
```

### Step 3: Grant Database Permissions

```sql
-- Admin role gets full access to all databases
GRANT USAGE, CREATE, MODIFY ON DATABASE customer_db TO ROLE admin_role;
GRANT USAGE, CREATE, MODIFY ON DATABASE analytics_db TO ROLE admin_role;

-- Engineer role gets create permissions
GRANT USAGE ON DATABASE customer_db TO ROLE engineer_role;
GRANT CREATE, MODIFY ON SCHEMA customer_db.public TO ROLE engineer_role;

-- Analyst role gets read-only access
GRANT USAGE ON DATABASE analytics_db TO ROLE analyst_role;
GRANT USAGE ON SCHEMA analytics_db.public TO ROLE analyst_role;

-- Viewer role gets very limited access
GRANT USAGE ON DATABASE analytics_db TO ROLE viewer_role;
```

### Step 4: Grant Table Permissions

```sql
-- Grant table-level permissions
GRANT SELECT ON TABLE customer_db.public.customers TO ROLE analyst_role;
GRANT SELECT ON TABLE customer_db.public.orders TO ROLE analyst_role;

-- Grant all tables in schema
GRANT SELECT ON ALL TABLES IN SCHEMA customer_db.public TO ROLE analyst_role;

-- Grant to specific columns only (using views)
CREATE VIEW customers_view AS
  SELECT customer_id, name, email FROM customers;

GRANT SELECT ON VIEW customers_view TO ROLE viewer_role;
```

### Step 5: Grant Warehouse Permissions

```sql
-- Allow roles to use specific warehouses
GRANT USAGE ON WAREHOUSE compute_wh TO ROLE admin_role;
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE analyst_role;
GRANT USAGE ON WAREHOUSE query_wh TO ROLE viewer_role;

-- Admin can also create warehouses
GRANT CREATE WAREHOUSE ON ACCOUNT TO ROLE admin_role;
```

### Step 6: Assign Roles to Users

```sql
-- Assign primary roles (used at login)
GRANT ROLE admin_role TO USER admin_user;
GRANT ROLE analyst_role TO USER analyst_user;
GRANT ROLE engineer_role TO USER engineer_user;
GRANT ROLE viewer_role TO USER viewer_user;

-- A user can have multiple roles
GRANT ROLE viewer_role TO USER analyst_user;

-- Verify role assignments
SHOW GRANTS TO USER analyst_user;
SHOW GRANTS ON USER analyst_user;
```

### Step 7: Hierarchy of Roles (Role Inheritance)

```sql
-- Create a parent role
CREATE ROLE data_team_role;

-- Create child roles that inherit from parent
GRANT ROLE analyst_role TO ROLE data_team_role;
GRANT ROLE engineer_role TO ROLE data_team_role;

-- Assign parent role to user
GRANT ROLE data_team_role TO USER team_lead_user;

-- Now team_lead_user has both analyst and engineer permissions
```

## Complete Role Setup Script

```sql
-- Complete role and user setup

-- 1. Create Roles
CREATE OR REPLACE ROLE admin_role;
CREATE OR REPLACE ROLE analyst_role;
CREATE OR REPLACE ROLE engineer_role;
CREATE OR REPLACE ROLE viewer_role;

-- 2. Create Users
CREATE OR REPLACE USER admin_user 
  PASSWORD = 'SecureAdminPass123!' 
  DEFAULT_WAREHOUSE = 'compute_wh'
  DEFAULT_ROLE = 'admin_role';

CREATE OR REPLACE USER analyst_user 
  PASSWORD = 'SecureAnalystPass456!' 
  DEFAULT_WAREHOUSE = 'analytics_wh'
  DEFAULT_ROLE = 'analyst_role';

-- 3. Grant Database Permissions
GRANT USAGE, CREATE ON DATABASE customer_db TO ROLE admin_role;
GRANT USAGE ON DATABASE customer_db TO ROLE analyst_role;

-- 4. Grant Table Permissions
GRANT SELECT ON ALL TABLES IN SCHEMA customer_db.public TO ROLE analyst_role;

-- 5. Grant Warehouse Permissions
GRANT USAGE ON WAREHOUSE compute_wh TO ROLE admin_role;
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE analyst_role;

-- 6. Assign Roles to Users
GRANT ROLE admin_role TO USER admin_user;
GRANT ROLE analyst_role TO USER analyst_user;
```

## Permission Levels

### Database Level Permissions

```sql
-- USAGE: Can see the database
-- CREATE: Can create objects in database
-- MODIFY: Can modify existing objects

GRANT USAGE ON DATABASE customer_db TO ROLE analyst_role;
GRANT USAGE, CREATE ON DATABASE customer_db TO ROLE engineer_role;
GRANT USAGE, CREATE, MODIFY ON DATABASE customer_db TO ROLE admin_role;
```

### Schema Level Permissions

```sql
GRANT USAGE ON SCHEMA customer_db.public TO ROLE analyst_role;
GRANT USAGE, CREATE ON SCHEMA customer_db.public TO ROLE engineer_role;
GRANT USAGE, CREATE, MODIFY ON SCHEMA customer_db.public TO ROLE admin_role;
```

### Table Level Permissions

```sql
-- Read permissions
GRANT SELECT ON TABLE customers TO ROLE analyst_role;

-- Write permissions
GRANT INSERT, UPDATE, DELETE ON TABLE customers TO ROLE engineer_role;

-- All permissions
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON TABLE customers TO ROLE admin_role;
```

### View Level Permissions

```sql
GRANT SELECT ON VIEW customer_summary TO ROLE analyst_role;
```

## Column-Level Access Control

```sql
-- Use views to restrict column access
CREATE VIEW public_customer_info AS
  SELECT 
    customer_id,
    name,
    country
    -- Exclude email and created_date
  FROM customers;

-- Grant access to view only
GRANT SELECT ON VIEW public_customer_info TO ROLE viewer_role;

-- Viewer can only see limited columns
SELECT * FROM public_customer_info; -- viewer sees only 3 columns
```

## Dynamic Data Masking (Column Masking)

```sql
-- Create masking policy
CREATE MASKING POLICY email_masking AS (email_col VARCHAR)
  RETURNS VARCHAR ->
    CASE
      WHEN CURRENT_ROLE() IN ('ADMIN_ROLE', 'ANALYST_ROLE')
        THEN email_col
      ELSE '***@***.***'
    END;

-- Apply masking policy to column
ALTER TABLE customers MODIFY COLUMN email 
  SET MASKING POLICY email_masking;

-- Now viewers see masked email
SELECT * FROM customers; -- email shows as ***@***.***
```

## Row-Level Access Control

```sql
-- Create row access policy
CREATE ROW ACCESS POLICY customer_region_policy AS (region_col VARCHAR)
  RETURNS BOOLEAN ->
    CASE
      WHEN CURRENT_ROLE() = 'ADMIN_ROLE' THEN TRUE
      WHEN REGION_COL = CURRENT_USER() THEN TRUE
      ELSE FALSE
    END;

-- Add customers with region
ALTER TABLE customers ADD COLUMN region VARCHAR(50);

-- Apply row access policy
ALTER TABLE customers ADD ROW ACCESS POLICY customer_region_policy ON (region);

-- Now users only see their region's data
SELECT * FROM customers; -- Only shows users's own region
```

## Auditing User Activity

```sql
-- View login history
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE USER_NAME = 'analyst_user'
  AND EVENT_TIMESTAMP >= DATEADD(DAY, -7, CURRENT_DATE())
ORDER BY EVENT_TIMESTAMP DESC;

-- View query history by user
SELECT 
  QUERY_ID,
  USER_NAME,
  QUERY_TEXT,
  EXECUTION_TIME,
  WAREHOUSE_NAME,
  EXECUTION_STATUS
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE USER_NAME = 'analyst_user'
  AND EXECUTION_DATE >= DATEADD(DAY, -30, CURRENT_DATE())
ORDER BY EXECUTION_TIME DESC;

-- View privilege grants
SHOW GRANTS TO USER analyst_user;
SHOW GRANTS TO ROLE analyst_role;
SHOW GRANTS ON SCHEMA customer_db.public;
```

## User Management Operations

### Update User Password

```sql
-- Change user password
ALTER USER analyst_user SET PASSWORD = 'NewSecurePassword123!';

-- Force password change at next login
ALTER USER analyst_user SET MUST_CHANGE_PASSWORD = TRUE;
```

### Update User Settings

```sql
ALTER USER analyst_user SET 
  DEFAULT_WAREHOUSE = 'new_warehouse'
  DEFAULT_ROLE = 'new_role'
  EMAIL = 'newemail@company.com';
```

### Disable/Enable User

```sql
-- Disable user account
ALTER USER analyst_user SET DISABLED = TRUE;

-- Enable user account
ALTER USER analyst_user SET DISABLED = FALSE;
```

### Drop User

```sql
-- Remove user from system
DROP USER analyst_user;
```

## Privilege Management

### Revoke Permissions

```sql
-- Revoke specific permission
REVOKE SELECT ON TABLE customers FROM ROLE analyst_role;

-- Revoke all permissions
REVOKE ALL ON DATABASE customer_db FROM ROLE analyst_role;

-- Revoke role from user
REVOKE ROLE analyst_role FROM USER analyst_user;
```

### Show All Granted Privileges

```sql
-- Show what roles a user has
SHOW GRANTS TO USER analyst_user;

-- Show what privileges a role has
SHOW GRANTS TO ROLE analyst_role;

-- Show who has access to a specific object
SHOW GRANTS ON TABLE customers;

-- Show all grants on a database
SHOW GRANTS ON DATABASE customer_db;
```

## Security Best Practices

### 1. Implement Least Privilege

```sql
-- Bad - Too much access
GRANT ALL PRIVILEGES ON ALL TABLES IN DATABASE customer_db TO ROLE analyst_role;

-- Good - Only needed access
GRANT SELECT ON TABLE customers TO ROLE analyst_role;
GRANT SELECT ON TABLE orders TO ROLE analyst_role;
```

### 2. Use Role Hierarchy

```sql
-- Organize roles by level
CREATE ROLE executive_role;
GRANT ROLE viewer_role TO ROLE executive_role;

CREATE ROLE manager_role;
GRANT ROLE analyst_role TO ROLE manager_role;

CREATE ROLE director_role;
GRANT ROLE manager_role TO ROLE director_role;
```

### 3. Audit Regularly

```sql
-- Create audit table
CREATE TABLE privilege_audit (
  audit_date DATE,
  user_name VARCHAR,
  role_name VARCHAR,
  permission_type VARCHAR,
  object_type VARCHAR,
  object_name VARCHAR
);

-- Log all privilege changes
INSERT INTO privilege_audit
SELECT 
  CURRENT_DATE(),
  USER_NAME,
  ROLE_NAME,
  PRIVILEGE,
  OBJECT_TYPE,
  OBJECT_NAME
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLE_GRANTS;
```

### 4. Use Service Accounts for Automated Tasks

```sql
-- Create service account for ETL processes
CREATE USER etl_service
  PASSWORD = 'VeryStrongRandomPassword123!@#'
  DEFAULT_WAREHOUSE = 'etl_wh'
  DEFAULT_ROLE = 'etl_role'
  DISABLED = FALSE;

-- Create ETL role with minimal permissions
CREATE ROLE etl_role;
GRANT USAGE ON DATABASE staging_db TO ROLE etl_role;
GRANT ALL ON SCHEMA staging_db.raw_data TO ROLE etl_role;
GRANT ROLE etl_role TO USER etl_service;
```

## Query Examples by Role

### Admin Role Capabilities

```sql
-- Admin can query all users
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.USERS;

-- Admin can view all warehouses
SHOW WAREHOUSES;

-- Admin can view all databases
SHOW DATABASES;
```

### Analyst Role Capabilities

```sql
-- Analyst can query authorized tables
SELECT customer_id, name, email FROM customers;

-- Analyst can create personal views
CREATE VIEW customer_summary AS
  SELECT 
    customer_id,
    name,
    COUNT(*) OVER (PARTITION BY customer_id) as records
  FROM customers;
```

### Viewer Role Capabilities

```sql
-- Viewer can only SELECT from assigned views
SELECT * FROM public_customer_info;

-- Viewer cannot create objects
-- CREATE TABLE test AS SELECT * FROM customers; -- ERROR
```

## Multi-Account Access (Delegated Role)

```sql
-- Create role for cross-account access
CREATE MANAGED ACCOUNT managed_account_user
  ADMIN_NAME = 'initial_admin'
  ADMIN_PASSWORD = 'SecurePass123!'
  TYPE = 'READER';

-- Grant permissions for shared data access
GRANT ROLE reader_account_role 
  TO ROLE managed_account_role;
```

## Next Steps

1. **Implement Row-Level Security:** Use row access policies
2. **Set Up Auditing:** Monitor user activities
3. **Automate Role Management:** Script role provisioning
4. **Create Service Accounts:** For automated tasks

## Learning Outcomes

✅ Create roles and users  
✅ Grant permissions at database/schema/table levels  
✅ Implement role hierarchy  
✅ Apply masking policies  
✅ Audit user activities  
✅ Follow security best practices  

## Related Use Cases

- **Use Case 24:** Implementing Role-Based Access Control (RBAC)
- **Use Case 32:** Dynamic Data Masking with Policies
- **Use Case 25:** Monitoring Queries and Resource Usage

---

**Last Updated:** February 18, 2026
