# Use Case 1: Creating Your First Database and Table

## Problem Description

As a data engineer or analyst starting with Snowflake, you need to understand the fundamental concepts of organizing data in a cloud data warehouse. You need to:

1. Create a database to store your data
2. Create a schema to organize tables logically
3. Create a table with proper data types and constraints
4. Understand the hierarchy of Snowflake objects

This is the foundational step before any data can be loaded or analyzed.

## Business Context

A company wants to start tracking customer information in Snowflake. They need to set up the basic infrastructure to store customer details like ID, name, email, and registration date.

## Solution

### Step 1: Create a Database

```sql
-- Create a database to hold all customer-related data
CREATE DATABASE customer_db;

-- Verify database creation
SHOW DATABASES;
```

**Explanation:** 
- `CREATE DATABASE` creates a new database that serves as a top-level container in Snowflake's object hierarchy
- Databases are case-insensitive by default and stored in uppercase

### Step 2: Create a Schema

```sql
-- Switch to the customer database
USE DATABASE customer_db;

-- Create a schema to organize related tables
CREATE SCHEMA public;

-- Verify schema creation
SHOW SCHEMAS;
```

**Explanation:**
- Schemas are logical containers within a database
- Multiple schemas can exist in one database for better organization
- `public` is a common naming convention for non-sensitive data

### Step 3: Create a Table with Proper Data Types

```sql
-- Create a customers table with appropriate data types
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_date DATE DEFAULT CURRENT_DATE(),
    updated_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Verify table structure
DESCRIBE TABLE customers;
-- or use SHOW COLUMNS
SHOW COLUMNS IN customers;
```

**Column Definitions:**

| Column | Data Type | Purpose | Constraints |
|--------|-----------|---------|-------------|
| customer_id | INT | Unique customer identifier | PRIMARY KEY |
| name | VARCHAR(100) | Customer full name | NOT NULL |
| email | VARCHAR(100) | Customer email address | NOT NULL |
| created_date | DATE | Account creation date | DEFAULT CURRENT_DATE() |
| updated_date | TIMESTAMP | Last update timestamp | DEFAULT CURRENT_TIMESTAMP() |

### Data Types Used

- **INT:** Integer numbers for IDs
- **VARCHAR(100):** Variable-length character strings up to 100 characters
- **DATE:** Date values (YYYY-MM-DD format)
- **TIMESTAMP:** Date and time with precision

### Step 4: Insert Sample Data

```sql
-- Insert sample customer records
INSERT INTO customers (customer_id, name, email, created_date)
VALUES
    (1, 'John Smith', 'john.smith@example.com', '2025-01-15'),
    (2, 'Sarah Johnson', 'sarah.johnson@example.com', '2025-01-20'),
    (3, 'Michael Brown', 'michael.brown@example.com', '2025-02-01');

-- Verify data insertion
SELECT * FROM customers;
```

**Output:**
```
CUSTOMER_ID | NAME            | EMAIL                       | CREATED_DATE | UPDATED_DATE
1           | John Smith      | john.smith@example.com      | 2025-01-15   | 2026-02-18 10:30:45.123
2           | Sarah Johnson   | sarah.johnson@example.com   | 2025-01-20   | 2026-02-18 10:30:45.456
3           | Michael Brown   | michael.brown@example.com   | 2025-02-01   | 2026-02-18 10:30:45.789
```

## Snowflake Object Hierarchy

```
┌─────────────────────────────────┐
│     SNOWFLAKE ACCOUNT           │
├─────────────────────────────────┤
│  DATABASE: customer_db          │
│  ├─ SCHEMA: public              │
│  │  ├─ TABLE: customers         │
│  │  ├─ TABLE: orders            │
│  │  └─ TABLE: products          │
│  └─ SCHEMA: analytics           │
│     ├─ TABLE: customer_summary  │
│     └─ TABLE: sales_metrics     │
│  DATABASE: finance_db           │
│  └─ SCHEMA: accounting          │
│     └─ TABLE: transactions      │
└─────────────────────────────────┘
```

## Complete Script

```sql
-- Complete script for setting up database and table

-- 1. Create Database
CREATE DATABASE customer_db;

-- 2. Switch to Database
USE DATABASE customer_db;

-- 3. Create Schema
CREATE SCHEMA public;

-- 4. Create Table
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_date DATE DEFAULT CURRENT_DATE(),
    updated_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- 5. Insert Sample Data
INSERT INTO customers (customer_id, name, email, created_date)
VALUES
    (1, 'John Smith', 'john.smith@example.com', '2025-01-15'),
    (2, 'Sarah Johnson', 'sarah.johnson@example.com', '2025-01-20'),
    (3, 'Michael Brown', 'michael.brown@example.com', '2025-02-01');

-- 6. Verify
SELECT * FROM customers;
```

## Key Concepts

### 1. **Database**
- Top-level object in Snowflake
- Contains schemas and tables
- Each database has its own set of objects

### 2. **Schema**
- Logical container within a database
- Organizes related tables and objects
- Enables better access control and organization

### 3. **Table**
- Stores actual data in rows and columns
- Defined by columns with specific data types
- Can have constraints (PRIMARY KEY, NOT NULL, etc.)

### 4. **Data Types**
- **Numeric:** INT, BIGINT, FLOAT, DECIMAL
- **String:** VARCHAR, CHAR, TEXT
- **Date/Time:** DATE, TIMESTAMP, TIME
- **Semi-structured:** VARIANT (for JSON), ARRAY, OBJECT

### 5. **Constraints**
- **PRIMARY KEY:** Uniquely identifies each row
- **NOT NULL:** Column must have a value
- **DEFAULT:** Provides default value if none supplied
- **UNIQUE:** Each value in column must be unique

## Advanced Configurations

### Create Table with Comments

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY COMMENT 'Unique customer identifier',
    name VARCHAR(100) NOT NULL COMMENT 'Customer full name',
    email VARCHAR(100) NOT NULL COMMENT 'Customer email address',
    created_date DATE DEFAULT CURRENT_DATE() COMMENT 'Account creation date',
    updated_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP() COMMENT 'Last update timestamp'
)
COMMENT = 'Master customer data table';
```

### Create Table with Data Retention

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_date DATE DEFAULT CURRENT_DATE()
)
DATA_RETENTION_TIME_IN_DAYS = 90;
```

### Create Temporary Table

```sql
CREATE TEMPORARY TABLE temp_customers (
    customer_id INT,
    name VARCHAR(100)
);
```

## Verification Commands

```sql
-- Show all databases
SHOW DATABASES;

-- Show all schemas in current database
SHOW SCHEMAS;

-- Show all tables in current schema
SHOW TABLES;

-- Describe table structure
DESCRIBE TABLE customers;

-- Show table comments
SHOW TABLES LIKE 'customers';

-- Get DDL statement
SELECT GET_DDL('TABLE', 'customers');
```

## Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Database already exists | Trying to create duplicate database | Use `CREATE OR REPLACE` or `CREATE IF NOT EXISTS` |
| Column name is reserved | Using SQL keywords as column names | Quote the column name: `"date"` or choose different name |
| Duplicate PRIMARY KEY | Inserting duplicate key values | Ensure uniqueness in data |
| NOT NULL constraint violation | Inserting NULL in NOT NULL column | Provide values for required columns |

## Next Steps

1. **Load Data:** Proceed to Use Case 2 - Loading Data from CSV Files
2. **Create More Tables:** Add related tables (orders, products)
3. **Define Relationships:** Use foreign keys to link tables
4. **Set Up Security:** Create roles and assign permissions (Use Case 4)

## Learning Outcomes

✅ Understand Snowflake's object hierarchy  
✅ Create databases and schemas  
✅ Design tables with appropriate data types  
✅ Implement basic constraints  
✅ Insert and verify data  

## Related Use Cases

- **Use Case 2:** Loading Data from CSV Files
- **Use Case 3:** Basic SELECT Queries
- **Use Case 4:** Using Snowflake Roles and Users
- **Use Case 6:** Creating Views for Simplified Access

---

**Last Updated:** February 18, 2026
