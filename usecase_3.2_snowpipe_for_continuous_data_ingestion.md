# Use Case 22: Snowpipe for Continuous Data Ingestion

## Problem Description

Snowpipe enables you to:

1. Automatically load data as files arrive
2. Use event-based triggering
3. Load incrementally without batch windows
4. Reduce data latency to seconds

## Business Context

A company receives data files continuously in S3 and needs near-real-time loading.

## Solution

```sql
-- Create pipe
CREATE PIPE customer_pipe AS
  COPY INTO customers
  FROM @s3_stage/customers/
  FILE_FORMAT = csv_format
  ON_ERROR = 'SKIP_FILE';

-- Enable pipe
ALTER PIPE customer_pipe SET PIPE_EXECUTION_PAUSED = FALSE;

-- View pipe status
SELECT * FROM TABLE(INFORMATION_SCHEMA.PIPE_STATUS(PIPE_NAME => 'customer_pipe'));

-- Monitor pipe history
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE PIPE_NAME = 'CUSTOMER_PIPE'
ORDER BY REFRESH_TIME DESC
LIMIT 10;
```

## Next Steps

1. **S3 Notifications:** Configure S3 to notify Snowpipe
2. **Error Handling:** Implement retry logic
3. **Monitoring:** Track pipe performance

## Learning Outcomes

✅ Create Snowpipe  
✅ Enable automatic loading  
✅ Monitor pipe status  

---

**Last Updated:** February 18, 2026
