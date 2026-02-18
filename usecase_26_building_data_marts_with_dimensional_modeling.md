# Use Case 26: Building Data Marts with Dimensional Modeling

## Problem Description

Data marts organize data for analytics using:

1. Fact tables (measurements)
2. Dimension tables (context)
3. Star schema (optimized for queries)
4. Conformed dimensions (shared across marts)

## Business Context

A company needs separate marts for:
- Finance (revenue analysis)
- Sales (pipeline analysis)  
- Marketing (campaign analysis)
All sharing common customer and product dimensions.

## Solution

```sql
-- Create dimension tables
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY,
    customer_id INT,
    name VARCHAR(100),
    country VARCHAR(50),
    segment VARCHAR(50),
    created_date DATE
);

CREATE TABLE dim_product (
    product_key INT PRIMARY KEY,
    product_id INT,
    name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2)
);

CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    date_value DATE,
    year INT,
    quarter INT,
    month INT,
    day INT,
    week INT
);

-- Create fact table
CREATE TABLE fact_sales (
    fact_id INT PRIMARY KEY,
    customer_key INT,
    product_key INT,
    date_key INT,
    sales_amount DECIMAL(15,2),
    quantity INT,
    FOREIGN KEY (customer_key) REFERENCES dim_customer,
    FOREIGN KEY (product_key) REFERENCES dim_product,
    FOREIGN KEY (date_key) REFERENCES dim_date
) CLUSTER BY (date_key, customer_key);

-- Populate dimensions
INSERT INTO dim_customer
SELECT 
    ROW_NUMBER() OVER (ORDER BY customer_id) AS customer_key,
    customer_id,
    name,
    country,
    'Standard' AS segment,
    created_date
FROM customers;

-- Create mart view
CREATE VIEW sales_mart AS
SELECT 
    d.date_value,
    d.year,
    d.quarter,
    dc.name AS customer_name,
    dc.country,
    dp.name AS product_name,
    dp.category,
    f.quantity,
    f.sales_amount
FROM fact_sales f
JOIN dim_customer dc ON f.customer_key = dc.customer_key
JOIN dim_product dp ON f.product_key = dp.product_key
JOIN dim_date d ON f.date_key = d.date_key;

-- Query mart
SELECT 
    year,
    quarter,
    country,
    category,
    SUM(sales_amount) AS revenue,
    COUNT(*) AS transaction_count
FROM sales_mart
GROUP BY year, quarter, country, category
ORDER BY year DESC, quarter DESC, revenue DESC;
```

## Next Steps

1. **Conformed Dimensions:** Share across multiple marts
2. **Slowly Changing:** Implement SCD for dimension history
3. **Aggregate Tables:** Pre-build common aggregations

## Learning Outcomes

✅ Design star schema  
✅ Create dimensions and facts  
✅ Build analytical views  
✅ Optimize for query performance  

---

**Last Updated:** February 18, 2026
