# Use Case 9: Using String Functions

## Problem Description

Data quality often depends on proper text handling. You need to:

1. Search within strings
2. Extract substrings
3. Transform case (uppercase/lowercase)
4. Trim whitespace
5. Replace text patterns
6. Concatenate strings

## Business Context

Customer data contains inconsistencies:
- Names have mixed case
- Emails have extra spaces
- Phone numbers have different formats
- Addresses need standardization

## Solution

### Case Transformation

```sql
CREATE TABLE customers_raw (
    customer_id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    city VARCHAR(100)
);

-- Case functions
SELECT 
    name,
    UPPER(name) AS uppercase_name,
    LOWER(name) AS lowercase_name,
    INITCAP(name) AS proper_case_name
FROM customers_raw
LIMIT 10;
```

### String Length and Position

```sql
-- String length
SELECT 
    name,
    LENGTH(name) AS name_length,
    email,
    LENGTH(email) AS email_length
FROM customers_raw
LIMIT 10;

-- Find position of character
SELECT 
    email,
    POSITION('@' IN email) AS at_symbol_position,
    POSITION('.' IN email) AS dot_position
FROM customers_raw
LIMIT 10;
```

### Substring Extraction

```sql
-- Extract portions of string
SELECT 
    email,
    SUBSTRING(email, 1, POSITION('@' IN email) - 1) AS username,
    SUBSTRING(email, POSITION('@' IN email) + 1) AS domain
FROM customers_raw
LIMIT 10;

-- Fixed length substring
SELECT 
    name,
    SUBSTRING(name, 1, 3) AS first_three_chars,
    SUBSTRING(name, -3) AS last_three_chars
FROM customers_raw
LIMIT 10;
```

### Trimming and Padding

```sql
-- Remove whitespace
SELECT 
    '  John Smith  ' AS original,
    TRIM('  John Smith  ') AS trimmed,
    LTRIM('  John Smith  ') AS left_trimmed,
    RTRIM('  John Smith  ') AS right_trimmed;

-- Padding with characters
SELECT 
    customer_id,
    LPAD(customer_id::VARCHAR, 5, '0') AS padded_id,
    RPAD(name, 20, '.') AS padded_name
FROM customers_raw
LIMIT 10;
```

### String Replacement

```sql
-- Replace text patterns
SELECT 
    email,
    REPLACE(email, 'example.com', 'newdomain.com') AS updated_email,
    REPLACE(UPPER(name), ' ', '_') AS name_for_filename
FROM customers_raw
LIMIT 10;

-- Replace with regex
SELECT 
    email,
    REGEXP_REPLACE(email, '[^a-zA-Z0-9.@_-]', '') AS cleaned_email
FROM customers_raw
LIMIT 10;
```

### String Concatenation

```sql
-- Concatenate strings
SELECT 
    customer_id,
    name,
    email,
    CONCAT(name, ' (', email, ')') AS name_with_email,
    name || ' <' || email || '>' AS formatted_contact
FROM customers_raw
LIMIT 10;

-- Concatenate multiple columns
SELECT 
    CONCAT_WS(', ', name, city, 'Customer') AS full_label
FROM customers_raw
LIMIT 10;
```

### String Splitting

```sql
-- Split string into array
SELECT 
    email,
    SPLIT_PART(email, '@', 1) AS username,
    SPLIT_PART(email, '@', 2) AS domain
FROM customers_raw
WHERE email LIKE '%@%'
LIMIT 10;

-- Split email domain parts
SELECT 
    email,
    SPLIT_PART(SPLIT_PART(email, '@', 2), '.', 1) AS domain_name,
    SPLIT_PART(SPLIT_PART(email, '@', 2), '.', 2) AS domain_extension
FROM customers_raw
WHERE email LIKE '%@%'
LIMIT 10;
```

### Pattern Matching

```sql
-- Pattern matching with LIKE
SELECT 
    email,
    CASE
        WHEN email LIKE '%@gmail.com' THEN 'Gmail'
        WHEN email LIKE '%@yahoo.com' THEN 'Yahoo'
        WHEN email LIKE '%@hotmail.com' THEN 'Hotmail'
        WHEN email LIKE '%@%.%' THEN 'Corporate'
        ELSE 'Other'
    END AS email_provider
FROM customers_raw
WHERE email IS NOT NULL
LIMIT 10;

-- Regex pattern matching
SELECT 
    email,
    CASE
        WHEN REGEXP_LIKE(email, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$') 
            THEN 'Valid'
        ELSE 'Invalid'
    END AS email_validity
FROM customers_raw
LIMIT 10;
```

### String Comparison

```sql
-- Case-sensitive comparison
SELECT 
    name,
    CASE
        WHEN name = 'John Smith' THEN 'Exact match'
        WHEN LOWER(name) = 'john smith' THEN 'Case-insensitive match'
        ELSE 'No match'
    END AS match_type
FROM customers_raw;

-- Similarity matching (Soundex, Metaphone)
SELECT 
    name1,
    name2,
    SOUNDEX(name1) = SOUNDEX(name2) AS soundex_match
FROM (
    SELECT 'Jon Smith' AS name1, 'John Smith' AS name2
) t;
```

## Complete String Processing Script

```sql
-- Comprehensive string cleanup

-- 1. Clean and standardize customer names
SELECT 
    customer_id,
    TRIM(INITCAP(LOWER(name))) AS cleaned_name
FROM customers_raw;

-- 2. Extract email components for analysis
SELECT 
    customer_id,
    email,
    LOWER(SPLIT_PART(email, '@', 1)) AS username,
    LOWER(SPLIT_PART(email, '@', 2)) AS email_domain,
    SPLIT_PART(SPLIT_PART(email, '@', 2), '.', 1) AS domain_name
FROM customers_raw
WHERE email IS NOT NULL;

-- 3. Categorize emails by domain
SELECT 
    SPLIT_PART(SPLIT_PART(email, '@', 2), '.', 1) AS domain,
    COUNT(*) AS customer_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM customers_raw
WHERE email IS NOT NULL
GROUP BY SPLIT_PART(SPLIT_PART(email, '@', 2), '.', 1)
ORDER BY customer_count DESC;

-- 4. Validate and flag data quality issues
SELECT 
    customer_id,
    name,
    email,
    city,
    CASE WHEN name IS NULL OR LENGTH(TRIM(name)) = 0 THEN 'Missing name' END AS issue1,
    CASE WHEN email NOT LIKE '%@%.%' THEN 'Invalid email' END AS issue2,
    CASE WHEN city IS NULL OR LENGTH(TRIM(city)) = 0 THEN 'Missing city' END AS issue3
FROM customers_raw
WHERE 
    (name IS NULL OR LENGTH(TRIM(name)) = 0) OR
    (email NOT LIKE '%@%.%') OR
    (city IS NULL OR LENGTH(TRIM(city)) = 0);
```

## String Functions Reference

| Function | Purpose | Example |
|----------|---------|---------|
| UPPER | Convert to uppercase | UPPER(name) |
| LOWER | Convert to lowercase | LOWER(name) |
| INITCAP | Proper case | INITCAP(name) |
| LENGTH | String length | LENGTH(name) |
| SUBSTRING | Extract substring | SUBSTRING(name, 1, 3) |
| POSITION | Find character position | POSITION('@' IN email) |
| TRIM | Remove whitespace | TRIM(name) |
| LTRIM | Left trim | LTRIM(name) |
| RTRIM | Right trim | RTRIM(name) |
| REPLACE | Replace text | REPLACE(text, old, new) |
| CONCAT | Concatenate strings | CONCAT(first, last) |
| SPLIT_PART | Split by delimiter | SPLIT_PART(text, ',', 1) |
| LIKE | Pattern match | text LIKE 'pattern%' |
| REGEXP_LIKE | Regex match | REGEXP_LIKE(text, 'pattern') |

## Real-World Examples

### Example 1: Phone Number Formatting

```sql
-- Standardize phone numbers
SELECT 
    customer_id,
    phone_raw,
    -- Remove non-numeric characters
    REGEXP_REPLACE(phone_raw, '[^0-9]', '') AS phone_digits,
    -- Format as (XXX) XXX-XXXX
    '(' || SUBSTRING(REGEXP_REPLACE(phone_raw, '[^0-9]', ''), 1, 3) || ') ' ||
    SUBSTRING(REGEXP_REPLACE(phone_raw, '[^0-9]', ''), 4, 3) || '-' ||
    SUBSTRING(REGEXP_REPLACE(phone_raw, '[^0-9]', ''), 7, 4) AS phone_formatted
FROM customers_raw;
```

### Example 2: Address Parsing

```sql
-- Parse full address into components
SELECT 
    customer_id,
    full_address,
    TRIM(SPLIT_PART(full_address, ',', 1)) AS street,
    TRIM(SPLIT_PART(full_address, ',', 2)) AS city,
    TRIM(SPLIT_PART(full_address, ',', 3)) AS state,
    TRIM(SPLIT_PART(full_address, ',', 4)) AS zip
FROM customers_raw;
```

### Example 3: Product Code Generation

```sql
-- Generate unique product codes
SELECT 
    product_id,
    name,
    CONCAT(
        UPPER(SUBSTRING(category, 1, 3)),
        '-',
        LPAD(product_id, 5, '0'),
        '-',
        UPPER(SUBSTRING(REPLACE(name, ' ', ''), 1, 2))
    ) AS product_code
FROM products;
```

## Next Steps

1. **Data Validation:** Use string functions for quality checks
2. **Data Cleaning:** Standardize formats across datasets
3. **Advanced Text Processing:** Use regex for complex patterns

## Learning Outcomes

✅ Transform string cases  
✅ Extract substrings and components  
✅ Search within strings  
✅ Replace text patterns  
✅ Validate string formats  
✅ Concatenate and parse strings  

## Related Use Cases

- **Use Case 3:** Basic SELECT Queries
- **Use Case 2:** Loading Data from CSV Files

---

**Last Updated:** February 18, 2026
