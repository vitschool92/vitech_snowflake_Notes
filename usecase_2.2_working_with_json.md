# Use Case 12: Working with Semi-Structured Data (JSON)

## Problem Description

Modern data often comes in JSON format. You need to:

1. Store and query JSON data
2. Extract specific fields from JSON
3. Flatten nested structures
4. Convert between JSON and relational format
5. Validate JSON data

## Business Context

APIs return JSON responses with nested structures. A company needs to:
- Parse user event data in JSON format
- Extract nested attributes
- Store flexible schemas
- Query JSON fields efficiently

## Solution

### VARIANT Data Type

```sql
-- VARIANT is Snowflake's JSON storage type
CREATE TABLE events (
    event_id INT,
    user_id INT,
    event_data VARIANT,
    event_timestamp TIMESTAMP
);

-- Insert JSON data
INSERT INTO events VALUES
(1, 101, PARSE_JSON('{"event_type": "click", "page": "/home", "duration": 45}'), CURRENT_TIMESTAMP()),
(2, 102, PARSE_JSON('{"event_type": "scroll", "page": "/products", "distance": 500}'), CURRENT_TIMESTAMP());
```

### Basic JSON Extraction

```sql
-- Extract JSON fields using colon notation
SELECT 
    event_id,
    user_id,
    event_data:event_type AS event_type,
    event_data:page AS page,
    event_data:duration AS duration
FROM events;

-- Cast to specific data types
SELECT 
    event_id,
    event_data:event_type::STRING AS event_type,
    event_data:page::VARCHAR AS page,
    event_data:duration::INT AS duration,
    event_data:timestamp::TIMESTAMP AS event_time
FROM events;

-- Result:
-- EVENT_ID | USER_ID | EVENT_TYPE | PAGE       | DURATION
-- 1        | 101     | "click"    | "/home"    | 45
-- 2        | 102     | "scroll"   | "/products"| 500
```

### Accessing Nested JSON

```sql
-- Nested JSON structure
INSERT INTO events VALUES
(3, 103, PARSE_JSON('{
    "event_type": "purchase",
    "user": {
        "id": 103,
        "name": "John Doe",
        "location": {"country": "USA", "city": "New York"}
    },
    "product": {"id": 5, "price": 99.99},
    "timestamp": "2025-02-18T10:30:00Z"
}'), CURRENT_TIMESTAMP());

-- Access nested fields with dot notation
SELECT 
    event_id,
    event_data:event_type::STRING AS event_type,
    event_data:user:name::STRING AS user_name,
    event_data:user:location:country::STRING AS country,
    event_data:user:location:city::STRING AS city,
    event_data:product:price::DECIMAL(10,2) AS price
FROM events
WHERE event_data:event_type = 'purchase';
```

### Array Handling in JSON

```sql
-- JSON with arrays
INSERT INTO events VALUES
(4, 104, PARSE_JSON('{
    "event_type": "batch_purchase",
    "items": [
        {"product_id": 1, "quantity": 2, "price": 50.00},
        {"product_id": 2, "quantity": 1, "price": 99.99},
        {"product_id": 3, "quantity": 3, "price": 25.00}
    ]
}'), CURRENT_TIMESTAMP());

-- Use FLATTEN to expand arrays
SELECT 
    e.event_id,
    e.event_data:event_type::STRING AS event_type,
    f.value:product_id::INT AS product_id,
    f.value:quantity::INT AS quantity,
    f.value:price::DECIMAL(10,2) AS price
FROM events e,
LATERAL FLATTEN(input => e.event_data:items) f
WHERE e.event_data:event_type = 'batch_purchase';

-- Result:
-- EVENT_ID | EVENT_TYPE      | PRODUCT_ID | QUANTITY | PRICE
-- 4        | batch_purchase  | 1          | 2        | 50.00
-- 4        | batch_purchase  | 2          | 1        | 99.99
-- 4        | batch_purchase  | 3          | 3        | 25.00
```

### FLATTEN Function

```sql
-- Flatten JSON structure into multiple rows
SELECT 
    event_id,
    key AS field_name,
    value AS field_value
FROM events,
LATERAL FLATTEN(input => event_data)
WHERE event_id = 1;

-- Recursive flattening
SELECT 
    event_id,
    key,
    value,
    path
FROM events,
LATERAL FLATTEN(input => event_data, recursive => TRUE)
WHERE event_id = 3
ORDER BY path;
```

### JSON Validation and Processing

```sql
-- Check if value is valid JSON
SELECT 
    event_id,
    TRY_PARSE_JSON(event_data) AS parsed_data,
    PARSE_JSON(event_data) AS strict_parsed
FROM events;

-- Validate JSON structure
SELECT 
    event_id,
    event_data,
    TRY_PARSE_JSON(event_data::VARCHAR) IS NOT NULL AS is_valid_json,
    TYPEOF(event_data) AS data_type
FROM events;

-- Check for specific field existence
SELECT 
    event_id,
    event_data HAS KEY 'event_type' AS has_event_type,
    event_data HAS KEY 'user' AS has_user,
    event_data HAS KEY 'product' AS has_product
FROM events;
```

## Complete JSON Processing Example

```sql
-- Comprehensive JSON processing

-- 1. Parse and normalize JSON event data
CREATE TABLE normalized_events AS
SELECT 
    event_id,
    user_id,
    event_data:event_type::STRING AS event_type,
    event_data:timestamp::TIMESTAMP AS event_time,
    event_data:page::VARCHAR AS page,
    event_data:user:name::VARCHAR AS user_name,
    event_data:user:location:country::VARCHAR AS country,
    event_data:product:price::DECIMAL(10,2) AS product_price,
    event_data
FROM events
WHERE event_data IS NOT NULL;

-- 2. Flatten array within JSON
WITH flattened_items AS (
    SELECT 
        event_id,
        event_data:event_type::STRING AS event_type,
        f.value:product_id::INT AS product_id,
        f.value:quantity::INT AS quantity,
        f.value:price::DECIMAL(10,2) AS price
    FROM events e,
    LATERAL FLATTEN(input => e.event_data:items) f
)
SELECT 
    event_id,
    event_type,
    product_id,
    quantity,
    price,
    quantity * price AS line_total
FROM flattened_items;

-- 3. Aggregate JSON data
SELECT 
    event_data:event_type::STRING AS event_type,
    COUNT(*) AS event_count,
    COUNT(DISTINCT user_id) AS unique_users,
    AVG(event_data:duration::INT) AS avg_duration,
    MAX(event_data:duration::INT) AS max_duration
FROM events
WHERE event_data:event_type IS NOT NULL
GROUP BY event_data:event_type;
```

## JSON Functions Reference

| Function | Purpose | Example |
|----------|---------|---------|
| PARSE_JSON | Convert string to JSON | PARSE_JSON('{"key": "value"}') |
| TRY_PARSE_JSON | Safe JSON parsing | TRY_PARSE_JSON(text) |
| JSON column:field | Extract JSON field | data:field_name |
| :: (cast) | Cast to type | data:field::INT |
| HAS KEY | Check field existence | data HAS KEY 'field' |
| FLATTEN | Expand arrays/objects | FLATTEN(data) |
| OBJECT_CONSTRUCT | Build JSON | OBJECT_CONSTRUCT('key', value) |
| ARRAY_CONSTRUCT | Build array | ARRAY_CONSTRUCT(val1, val2) |

## Converting JSON to Relational Format

```sql
-- Decompose JSON into relational tables
CREATE TABLE events_normalized (
    event_id INT,
    user_id INT,
    event_type VARCHAR,
    page VARCHAR,
    duration INT,
    timestamp TIMESTAMP
);

INSERT INTO events_normalized
SELECT 
    event_id,
    user_id,
    event_data:event_type::VARCHAR,
    event_data:page::VARCHAR,
    event_data:duration::INT,
    event_data:timestamp::TIMESTAMP
FROM events
WHERE event_data:event_type IS NOT NULL;

-- Query normalized data
SELECT * FROM events_normalized;
```

## Next Steps

1. **Stream JSON Data:** Use Snowpipe for continuous JSON ingestion
2. **Complex Transformations:** Transform nested JSON at scale
3. **API Integration:** Load JSON from REST APIs

## Learning Outcomes

✅ Store JSON in VARIANT columns  
✅ Extract nested fields  
✅ Flatten arrays and objects  
✅ Validate JSON data  
✅ Convert JSON to relational format  

## Related Use Cases

- **Use Case 2:** Loading Data from CSV Files
- **Use Case 22:** Snowpipe for Continuous Data Ingestion
- **Use Case 33:** Advanced JSON Processing and Normalization

---

**Last Updated:** February 18, 2026
