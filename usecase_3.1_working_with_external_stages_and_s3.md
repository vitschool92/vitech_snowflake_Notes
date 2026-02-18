# Use Case 21: Working with External Stages and S3

## Problem Description

External stages enable you to:

1. Reference cloud storage
2. Load data from S3, Azure Blob, GCS
3. Unload data to cloud storage
4. Query files directly in stage

## Business Context

A company stores data files in S3 and needs to load/unload data efficiently.

## Solution

### Create External Stage

```sql
-- Create S3 stage with credentials
CREATE STAGE s3_data_stage
  URL = 's3://my-bucket/data/'
  CREDENTIALS = (AWS_KEY_ID = 'xxx' AWS_SECRET_KEY = 'yyy');

-- OR use IAM role (recommended)
CREATE STORAGE INTEGRATION s3_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::ACCOUNT-ID:role/snowflake-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://my-bucket/');

CREATE STAGE s3_stage
  URL = 's3://my-bucket/'
  STORAGE_INTEGRATION = s3_integration;

-- List files in stage
LIST @s3_stage;

-- Query files directly
SELECT * FROM @s3_stage/data.csv (FILE_FORMAT => 'csv_format') LIMIT 10;
```

### Load from Stage

```sql
-- Load data from S3
COPY INTO customers
  FROM @s3_stage/customers.csv
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
  ON_ERROR = 'CONTINUE';

-- Load with validation
COPY INTO customers
  FROM @s3_stage/customers/
  FILE_FORMAT = csv_format
  PATTERN = '.*customers.*\.csv'
  VALIDATION_MODE = 'RETURN_ERRORS';
```

### Unload to Stage

```sql
-- Unload data to S3
COPY (
    SELECT * FROM customers WHERE status = 'ACTIVE'
)
TO @s3_stage/active_customers/
FILE_FORMAT = (TYPE = 'PARQUET')
OVERWRITE = TRUE;

-- Export with compression
COPY (
    SELECT * FROM orders
)
TO @s3_stage/orders_export/
FILE_FORMAT = (TYPE = 'CSV' COMPRESSION = 'GZIP')
SINGLE = FALSE
MAX_FILE_SIZE = 52428800;  -- 50 MB files
```

## Next Steps

1. **Automated Unload:** Schedule regular exports
2. **External Tables:** Query S3 data directly
3. **Data Exchange:** Share data via S3

## Learning Outcomes

✅ Create external stages  
✅ Load from cloud storage  
✅ Unload to cloud storage  
✅ Use IAM roles  

---

**Last Updated:** February 18, 2026
