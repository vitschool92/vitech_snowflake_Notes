# Use Case 16: Using Clustering Keys for Performance

## Problem Description

Large tables require optimization for query performance. Clustering keys:

1. Physically organize data based on columns
2. Reduce data scanned by queries
3. Speed up filtered queries
4. Lower query costs
5. Improve memory efficiency

## Business Context

A company has billions of sales records by date and region. Queries often filter by both dimensions, so clustering on these columns improves performance significantly.

## Solution

### Creating Clustered Tables

```sql
-- Create table with clustering key
CREATE TABLE sales_data (
    sales_id INT,
    region VARCHAR(50),
    product_id INT,
    sale_date DATE,
    amount DECIMAL(10,2),
    customer_id INT
)
CLUSTER BY (region, sale_date);

-- Verify clustering
SHOW TABLES LIKE 'sales_data';
-- CLUSTER_KEY column shows: region, sale_date
```

### Clustering Best Practices

```sql
-- Good: Cluster on frequently filtered columns
CREATE TABLE transactions (
    transaction_id INT,
    user_id INT,
    transaction_date DATE,
    amount DECIMAL(10,2),
    category VARCHAR(50)
)
CLUSTER BY (user_id, transaction_date);

-- Query benefits from clustering
SELECT * FROM transactions
WHERE user_id = 12345
  AND transaction_date >= '2025-01-01';

-- Add clustering to existing table
ALTER TABLE transactions
CLUSTER BY (user_id, transaction_date);
```

### Reclustering

```sql
-- Manually recluster when needed
ALTER TABLE sales_data RECLUSTER;

-- Monitor clustering effectiveness
SELECT 
    TABLE_NAME,
    CLUSTERING_KEY,
    BYTES_COMPRESSED,
    BYTES_UNCOMPRESSED,
    BYTE_RATIO
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE TABLE_NAME = 'SALES_DATA';
```

## Clustering Metrics and Monitoring

```sql
-- Check clustering effectiveness
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.CLUSTERING_INFORMATION
WHERE TABLE_NAME = 'SALES_DATA'
  AND TABLE_SCHEMA = 'PUBLIC';

-- Query performance impact
SELECT 
    TABLE_NAME,
    CLUSTERING_KEY,
    TOTAL_BYTES_SCANNED,
    ROWS_READ,
    ROWS_RETURNED,
    SCAN_EFFICIENCY
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_STATISTICS
WHERE TABLE_NAME = 'SALES_DATA';
```

## Next Steps

1. **Partition Pruning:** Combine with other optimization techniques
2. **Multi-Column Clustering:** Optimize complex queries
3. **Monitoring:** Track clustering effectiveness

## Learning Outcomes

✅ Understand clustering benefits  
✅ Choose optimal clustering columns  
✅ Monitor clustering performance  
✅ Recluster tables  

## Related Use Cases

- **Use Case 28:** Advanced Data Partitioning Strategies
- **Use Case 42:** Advanced Query Optimization

---

**Last Updated:** February 18, 2026
