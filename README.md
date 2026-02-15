/* =====================================================
  PORTFOLIO PROJECT
   =====================================================
   Database  : Portfolio
   Author    : Komal Maurya
   Purpose   : Data warehouse creation + analytics
   ===================================================== */

-- =====================================================
-- DATABASE
-- =====================================================
CREATE DATABASE Portfolio;
GO

USE Portfolio;
GO

-- =====================================================
-- TABLES
-- =====================================================

CREATE TABLE dim_customers(
    customer_key INT,
    customer_id INT,
    customer_number NVARCHAR(50),
    first_name NVARCHAR(50),
    last_name NVARCHAR(50),
    country NVARCHAR(50),
    marital_status NVARCHAR(50),
    gender NVARCHAR(50),
    birthdate DATE,
    create_date DATE
);

CREATE TABLE dim_products(
    product_key INT,
    product_id INT,
    product_number NVARCHAR(50),
    product_name NVARCHAR(50),
    category_id NVARCHAR(50),
    category NVARCHAR(50),
    subcategory NVARCHAR(50),
    maintenance NVARCHAR(50),
    cost INT,
    product_line NVARCHAR(50),
    start_date DATE
);

CREATE TABLE fact_sales(
    order_number NVARCHAR(50),
    product_key INT,
    customer_key INT,
    order_date DATE,
    shipping_date DATE,
    due_date DATE,
    sales_amount INT,
    quantity TINYINT,
    price INT
);

-- =====================================================
-- DATA IMPORT
-- Replace file paths as per your system
-- =====================================================

BULK INSERT dim_customers
FROM 'data/dim_customers.csv'
WITH (
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    TABLOCK
);

BULK INSERT dim_products
FROM 'data/dim_products.csv'
WITH (
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    TABLOCK
);

BULK INSERT fact_sales
FROM 'data/fact_sales.csv'
WITH (
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    TABLOCK
);

-- =====================================================
-- BASIC CHECK
-- =====================================================
SELECT * FROM dim_customers;
SELECT * FROM dim_products;
SELECT * FROM fact_sales;

-- =====================================================
-- 1. SALES TREND OVER TIME
-- =====================================================

SELECT 
    YEAR(order_date) AS order_year,
    MONTH(order_date) AS order_month,
    SUM(sales_amount) AS total_sales,
    COUNT(DISTINCT customer_key) AS total_customers,
    SUM(quantity) AS total_quantity
FROM fact_sales
WHERE order_date IS NOT NULL
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY order_year, order_month;

-- =====================================================
-- 2. CUMULATIVE SALES ANALYSIS
-- =====================================================

WITH yearly_sales AS (
    SELECT 
        YEAR(order_date) AS order_year,
        SUM(sales_amount) AS total_sales,
        AVG(price) AS avg_price
    FROM fact_sales
    WHERE order_date IS NOT NULL
    GROUP BY YEAR(order_date)
)

SELECT 
    order_year,
    total_sales,
    SUM(total_sales) OVER (ORDER BY order_year) AS running_total_sales,
    AVG(avg_price) OVER (ORDER BY order_year) AS moving_avg_price
FROM yearly_sales;

-- =====================================================
-- 3. PRODUCT PERFORMANCE (YoY + vs Average)
-- =====================================================

WITH yearly_product_sales AS (
    SELECT
        YEAR(f.order_date) AS order_year,
        p.product_name,
        SUM(f.sales_amount) AS total_current_sales
    FROM fact_sales f
    LEFT JOIN dim_products p
        ON f.product_key = p.product_key
    WHERE f.order_date IS NOT NULL
    GROUP BY YEAR(f.order_date), p.product_name
)

SELECT 
    order_year,
    product_name,
    total_current_sales,
    AVG(total_current_sales) OVER (PARTITION BY product_name) AS avg_sales,
    total_current_sales 
        - AVG(total_current_sales) OVER (PARTITION BY product_name) AS diff_from_avg,
    LAG(total_current_sales) OVER (
        PARTITION BY product_name ORDER BY order_year
    ) AS previous_year_sales
FROM yearly_product_sales
ORDER BY product_name, order_year;

-- =====================================================
-- 4. CATEGORY CONTRIBUTION
-- =====================================================

WITH category_sales AS (
    SELECT 
        p.category,
        SUM(f.sales_amount) AS total_sales
    FROM fact_sales f
    LEFT JOIN dim_products p
        ON p.product_key = f.product_key
    GROUP BY p.category
)

SELECT 
    category,
    total_sales,
    SUM(total_sales) OVER () AS overall_sales,
    ROUND(
        CAST(total_sales AS FLOAT) /
        SUM(total_sales) OVER () * 100, 2
    ) AS percentage_of_total
FROM category_sales;

-- =====================================================
-- 5. PRODUCT COST SEGMENTATION
-- =====================================================

WITH product_segments AS (
    SELECT
        product_key,
        CASE 
            WHEN cost < 100 THEN 'Below 100'
            WHEN cost BETWEEN 100 AND 500 THEN '100-500'
            WHEN cost BETWEEN 500 AND 1000 THEN '500-1000'
            ELSE 'Above 1000'
        END AS cost_range
    FROM dim_products
)

SELECT 
    cost_range,
    COUNT(product_key) AS total_products
FROM product_segments
GROUP BY cost_range
ORDER BY total_products DESC;

-- =====================================================
-- 6. CUSTOMER SEGMENTATION
-- =====================================================

WITH customer_spending AS (
    SELECT 
        c.customer_key,
        SUM(f.sales_amount) AS total_spending,
        DATEDIFF(MONTH, MIN(order_date), MAX(order_date)) AS lifespan
    FROM fact_sales f
    LEFT JOIN dim_customers c
        ON f.customer_key = c.customer_key
    GROUP BY c.customer_key
)

SELECT 
    CASE 
        WHEN lifespan >= 12 AND total_spending > 5000 THEN 'VIP'
        WHEN lifespan >= 12 AND total_spending <= 5000 THEN 'Regular'
        ELSE 'New'
    END AS customer_segment,
    COUNT(customer_key) AS total_customers
FROM customer_spending
GROUP BY 
    CASE 
        WHEN lifespan >= 12 AND total_spending > 5000 THEN 'VIP'
        WHEN lifespan >= 12 AND total_spending <= 5000 THEN 'Regular'
        ELSE 'New'
    END
ORDER BY total_customers DESC;

-- =====================================================
-- 7. CUSTOMER REPORT VIEW
-- =====================================================

CREATE VIEW report_customers AS

WITH base_query AS (
    SELECT 
        f.order_number,
        f.product_key,
        f.order_date,
        f.sales_amount,
        f.quantity,
        c.customer_key,
        c.customer_number,
        CONCAT(c.first_name,' ',c.last_name) AS customer_name,
        DATEDIFF(YEAR, c.birthdate, GETDATE()) AS age
    FROM fact_sales f
    LEFT JOIN dim_customers c
        ON c.customer_key = f.customer_key
    WHERE order_date IS NOT NULL
),

customer_aggregation AS (
    SELECT 
        customer_key,
        customer_number,
        customer_name,
        age,
        COUNT(DISTINCT order_number) AS total_orders,
        SUM(quantity) AS total_qty,
        SUM(sales_amount) AS total_sales,
        MAX(order_date) AS last_order_date,
        DATEDIFF(MONTH, MIN(order_date), MAX(order_date)) AS lifespan
    FROM base_query
    GROUP BY customer_key, customer_number, customer_name, age
)

SELECT 
    *,
    DATEDIFF(MONTH, last_order_date, GETDATE()) AS recency,
    CASE WHEN total_orders = 0 THEN 0
         ELSE total_sales / total_orders END AS avg_order_value
FROM customer_aggregation;

-- To use:
SELECT * FROM report_customers;
