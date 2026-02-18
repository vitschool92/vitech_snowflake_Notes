# Use Case 2: Loading Data from CSV Files

## Problem Description

After setting up your database and tables, you need to load data from CSV files into Snowflake. Common challenges include:

1. Managing file storage and access credentials
2. Understanding different file formats and their specifications
3. Handling data quality issues during loading
4. Validating that data was loaded correctly
5. Implementing error handling for failed loads

This is a critical operation for data warehousing workflows.

## Business Context

A company has customer data in CSV files stored in an AWS S3 bucket. They need to:
- Load customer information regularly
- Handle CSV files with headers
- Validate data quality
- Track loading success/failures

## Solution

### Step 1: Create an S3 Stage

A stage in Snowflake is a location pointer to cloud storage where files are kept.

```sql
-- Create an external stage pointing to S3 bucket
CREATE STAGE my_stage
  URL = 's3://my-bucket/path/'
  CREDENTIALS = (AWS_KEY_ID = 'XXXXXXXXXX' AWS_SECRET_KEY = 'XXXXXXXXXX');

-- Verify stage creation
SHOW STAGES;

-- List files in the stage
LIST @my_stage;
```

**Important:** Replace credentials with actual AWS credentials

### Step 2: Create a File Format

Define how Snowflake should parse your CSV files:

```sql
-- Create a CSV file format
CREATE OR REPLACE FILE FORMAT csv_format
  TYPE = 'CSV'
  COMPRESSION = 'AUTO'
  FIELD_DELIMITER = ','
  RECORD_DELIMITER = '\n'
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE
  NULL_IF = ('NULL', '');

-- Verify file format
SHOW FILE FORMATS;

-- Describe file format
DESCRIBE FILE FORMAT csv_format;
```

**File Format Options Explained:**

| Option | Value | Explanation |
|--------|-------|-------------|
| TYPE | CSV | Specifies CSV format |
| COMPRESSION | AUTO | Automatically detect compression |
| FIELD_DELIMITER | , | Column separator |
| RECORD_DELIMITER | \n | Row separator |
| SKIP_HEADER | 1 | Skip first header row |
| FIELD_OPTIONALLY_ENCLOSED_BY | " | Handle quoted fields |
| ERROR_ON_COLUMN_COUNT_MISMATCH | TRUE | Fail if column count doesn't match |
| NULL_IF | ('NULL', '') | Treat these as NULL values |

### Step 3: Prepare Sample CSV File

Create a sample CSV file (`customers.csv`):

```
customer_id,name,email,created_date
1,John Smith,john.smith@example.com,2025-01-15
2,Sarah Johnson,sarah.johnson@example.com,2025-01-20
3,Michael Brown,michael.brown@example.com,2025-02-01
4,Emily Davis,emily.davis@example.com,2025-02-05
5,David Wilson,david.wilson@example.com,2025-02-10
```

### Step 4: Load Data Using COPY Command

```sql
-- Basic COPY command to load data
COPY INTO customers
  FROM @my_stage/customers.csv
  FILE_FORMAT = csv_format;

-- View the result
SELECT * FROM customers;
```

### Step 5: Advanced COPY with Error Handling

```sql
-- COPY with error tolerance and validation
COPY INTO customers
  FROM @my_stage/customers.csv
  FILE_FORMAT = csv_format
  ON_ERROR = 'CONTINUE'
  SIZE_LIMIT = 10000000
  PURGE = FALSE;

-- Check loading errors
SELECT * FROM TABLE(VALIDATE(customers, JOB_ID => 'LAST'));
```

**COPY Parameters:**

| Parameter | Value | Explanation |
|-----------|-------|-------------|
| ON_ERROR | CONTINUE | Continue loading on error (vs ABORT) |
| SIZE_LIMIT | bytes | Maximum total size to load |
| PURGE | FALSE | Keep files after load (TRUE = delete) |
| MATCH_BY_COLUMN_NAME | CASE_INSENSITIVE | Match by column name instead of position |
| FORCE | TRUE | Reload files even if already loaded |

### Step 6: Create a Loading History Table

Track all data loads for auditing:

```sql
-- Create loading history table
CREATE TABLE load_history (
    load_id INT AUTOINCREMENT,
    file_name VARCHAR(500),
    table_name VARCHAR(100),
    load_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    rows_loaded INT,
    rows_failed INT,
    status VARCHAR(50),
    error_message VARCHAR(1000)
);

-- Log successful load
INSERT INTO load_history (file_name, table_name, rows_loaded, rows_failed, status)
VALUES ('customers.csv', 'customers', 5, 0, 'SUCCESS');
```

## Complete Loading Script

```sql
-- Complete data loading script

-- 1. Create Stage
CREATE STAGE IF NOT EXISTS my_stage
  URL = 's3://my-bucket/path/'
  CREDENTIALS = (AWS_KEY_ID = 'XXXXXXXXXX' AWS_SECRET_KEY = 'XXXXXXXXXX');

-- 2. Create File Format
CREATE OR REPLACE FILE FORMAT csv_format
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('NULL', '');

-- 3. Create Target Table (if not exists)
CREATE TABLE IF NOT EXISTS customers (
    customer_id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    created_date DATE
);

-- 4. Load Data
COPY INTO customers
  FROM @my_stage/customers.csv
  FILE_FORMAT = csv_format
  ON_ERROR = 'CONTINUE';

-- 5. Verify Load
SELECT COUNT(*) AS total_records FROM customers;
SELECT * FROM customers LIMIT 10;
```

## Alternative Loading Methods

### Method 1: Using PUT Command (Internal Stage)

```sql
-- Create internal stage
CREATE STAGE internal_stage;

-- Upload file from local machine
PUT file:///path/to/customers.csv @internal_stage;

-- Load from internal stage
COPY INTO customers
  FROM @internal_stage/customers.csv
  FILE_FORMAT = csv_format;
```

### Method 2: Direct Loading from URL

```sql
-- If your CSV is publicly accessible via URL
COPY INTO customers
  FROM 'https://example.com/data/customers.csv'
  FILE_FORMAT = csv_format;
```

### Method 3: Using Snowpipe for Continuous Loading

```sql
-- Create Snowpipe for automatic loading
CREATE PIPE customer_pipe AS
  COPY INTO customers
  FROM @my_stage/customers/
  FILE_FORMAT = csv_format
  ON_ERROR = 'SKIP_FILE';

-- Resume pipe to enable it
ALTER PIPE customer_pipe SET PIPE_EXECUTION_PAUSED = FALSE;

-- Monitor Snowpipe status
SELECT * FROM TABLE(INFORMATION_SCHEMA.PIPE_STATUS(PIPE_NAME => 'customer_pipe'));
```

## Error Handling and Validation

### Check Load Status

```sql
-- View copy history
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE TABLE_NAME = 'CUSTOMERS'
ORDER BY LAST_LOAD_TIME DESC
LIMIT 10;
```

### Handle Data Quality Issues

```sql
-- Create staging table for validation
CREATE TABLE customers_staging (
    customer_id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    created_date DATE,
    load_error VARCHAR(500)
);

-- Load with error tracking
COPY INTO customers_staging
  FROM @my_stage/customers.csv
  FILE_FORMAT = csv_format
  ON_ERROR = 'CONTINUE';

-- Identify and fix issues
SELECT * FROM customers_staging 
WHERE load_error IS NOT NULL;

-- Move validated data to main table
INSERT INTO customers
SELECT customer_id, name, email, created_date
FROM customers_staging
WHERE load_error IS NULL;
```

### Data Validation Queries

```sql
-- Check for duplicates
SELECT customer_id, COUNT(*) as count
FROM customers
GROUP BY customer_id
HAVING COUNT(*) > 1;

-- Check for null values in required fields
SELECT * FROM customers
WHERE customer_id IS NULL OR name IS NULL;

-- Check date format validity
SELECT * FROM customers
WHERE created_date > CURRENT_DATE();

-- Check email format
SELECT * FROM customers
WHERE email NOT LIKE '%@%.%';
```

## Advanced: Multi-File Loading

```sql
-- Load multiple CSV files from S3 path
COPY INTO customers
  FROM @my_stage/customers/
  FILE_FORMAT = csv_format
  PATTERN = '.*customers_.*\.csv'
  ON_ERROR = 'CONTINUE';

-- Load with file format detection
COPY INTO customers
  FROM @my_stage/customers/
  FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1)
  ON_ERROR = 'SKIP_FILE';
```

## Performance Considerations

### Parallel Loading

```sql
-- Snowflake automatically parallelizes large loads
-- For files > 100MB, Snowflake splits them automatically

-- To optimize, upload multiple files and load in parallel
-- File size recommendations: 50-100 MB per file

-- Monitor warehouse utilization during load
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE WAREHOUSE_NAME = 'YOUR_WH'
ORDER BY START_TIME DESC
LIMIT 10;
```

### Warehouse Sizing

```sql
-- Use larger warehouse for initial bulk loads
CREATE WAREHOUSE load_wh
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 300;

-- Switch to load warehouse
USE WAREHOUSE load_wh;

-- Perform load
COPY INTO customers FROM @my_stage/customers.csv FILE_FORMAT = csv_format;

-- Switch back to normal warehouse
USE WAREHOUSE compute_wh;
```

## Security Best Practices

### Using External ID for S3 Access

```sql
-- Create storage integration for secure access
CREATE STORAGE INTEGRATION s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::ACCOUNT-ID:role/snowflake-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://my-bucket/path/');

-- Use storage integration in stage
CREATE STAGE my_stage
  URL = 's3://my-bucket/path/'
  STORAGE_INTEGRATION = s3_int;
```

### Masking Sensitive Data

```sql
-- Create masking policy for sensitive columns
CREATE MASKING POLICY email_masking AS (email VARCHAR)
RETURNS VARCHAR ->
    CASE
        WHEN CURRENT_ROLE() = 'DATA_ENGINEER' THEN email
        ELSE '***@***.***'
    END;

-- Apply to column
ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY email_masking;
```

## Troubleshooting Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "File not found" | Stage path incorrect | Verify with `LIST @stage_name` |
| "Column count mismatch" | CSV columns != table columns | Check file format and table structure |
| "Invalid date format" | Date values don't match expected format | Ensure dates are YYYY-MM-DD format |
| "Access denied" | Bad AWS credentials | Verify IAM permissions and keys |
| "Out of memory" | File too large | Split into smaller files or increase warehouse size |

## Monitoring and Auditing

```sql
-- Create audit log table
CREATE TABLE audit_log (
    log_id INT AUTOINCREMENT,
    operation VARCHAR(50),
    table_name VARCHAR(100),
    user_name VARCHAR(100),
    operation_timestamp TIMESTAMP,
    rows_affected INT,
    status VARCHAR(50)
);

-- Log all load operations
INSERT INTO audit_log (operation, table_name, user_name, rows_affected, status)
SELECT 
    'COPY',
    'customers',
    CURRENT_USER(),
    rows_loaded,
    'SUCCESS'
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE TABLE_NAME = 'CUSTOMERS'
  AND LAST_LOAD_TIME > DATEADD(HOUR, -1, CURRENT_TIMESTAMP());
```

## Next Steps

1. **Automate Loading:** Use Snowpipe for continuous data ingestion
2. **Transform Data:** Apply transformations after loading
3. **Create Views:** Build views for data consumers
4. **Schedule Tasks:** Automate loading on a schedule

## Learning Outcomes

✅ Create and manage stages in Snowflake  
✅ Define file formats for data parsing  
✅ Load data using COPY command  
✅ Implement error handling and validation  
✅ Monitor and audit data loads  

## Related Use Cases

- **Use Case 1:** Creating Your First Database and Table
- **Use Case 22:** Snowpipe for Continuous Data Ingestion
- **Use Case 31:** Building ETL Pipelines with Tasks and Streams
- **Use Case 47:** Change Data Capture (CDC) Pipeline

---

**Last Updated:** February 18, 2026
