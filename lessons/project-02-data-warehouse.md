# Project 2: Data Warehouse Implementation

> **Capstone Project • Assessment 2**  
> Estimated time: 5-7 hours | Difficulty: ★★★★★

## 1. Project Overview

Design and implement a complete data warehouse solution for business intelligence and analytics. This project focuses on OLAP (Online Analytical Processing) workloads, requiring you to transform transactional data into analytical structures optimized for reporting and decision-making.

You'll build a dimensional model, implement ETL processes, create complex analytical queries, and set up automated reporting pipelines that could support a real business intelligence team.

**Learning Objectives:**
- Design star and snowflake schemas for analytical workloads
- Implement ETL/ELT processes for data transformation
- Create complex analytical queries with window functions
- Build automated reporting and data quality monitoring
- Optimize for large-scale analytical workloads

> **Prerequisites**: Complete [Project 1: E-commerce Database](project-01-ecommerce-database.md) or have equivalent OLTP data source.

## 2. Business Context

You're building a data warehouse for "TechMart Analytics" - the business intelligence arm of the e-commerce company from Project 1. The executive team needs:

### Analytical Requirements
- **Sales Analytics**: Revenue trends, product performance, customer segmentation
- **Inventory Analytics**: Stock levels, turnover rates, demand forecasting
- **Customer Analytics**: Lifetime value, cohort analysis, churn prediction
- **Marketing Analytics**: Campaign effectiveness, channel attribution
- **Operational Analytics**: Order fulfillment, shipping performance, returns

### Reporting Needs
- Daily/weekly/monthly executive dashboards
- Ad-hoc analytical queries by business analysts
- Historical trend analysis (3+ years of data)
- Real-time operational metrics
- Regulatory compliance reporting

### Performance Requirements
- Sub-second response for dashboard queries
- Support for complex analytical workloads
- Handle 100GB+ of historical data
- Process daily data loads within 2-hour window
- Support 50+ concurrent analytical users

## 3. Phase 1: Dimensional Modeling

### 3.1 Star Schema Design

Design fact and dimension tables following Kimball methodology:

```sql
-- Create the data warehouse database
CREATE DATABASE techmart_dw
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

USE techmart_dw;

-- Date Dimension (crucial for time-based analysis)
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE NOT NULL,
    day_of_week TINYINT NOT NULL,
    day_name VARCHAR(10) NOT NULL,
    day_of_month TINYINT NOT NULL,
    day_of_year SMALLINT NOT NULL,
    week_of_year TINYINT NOT NULL,
    month_number TINYINT NOT NULL,
    month_name VARCHAR(10) NOT NULL,
    quarter TINYINT NOT NULL,
    quarter_name VARCHAR(2) NOT NULL,
    year SMALLINT NOT NULL,
    is_weekend BOOLEAN NOT NULL,
    is_holiday BOOLEAN DEFAULT FALSE,
    holiday_name VARCHAR(50) NULL,
    fiscal_year SMALLINT NOT NULL,
    fiscal_quarter TINYINT NOT NULL,
    
    INDEX idx_full_date (full_date),
    INDEX idx_year_month (year, month_number),
    INDEX idx_quarter (fiscal_year, fiscal_quarter)
);

-- Customer Dimension (Type 2 SCD - Slowly Changing Dimension)
CREATE TABLE dim_customer (
    customer_key INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL, -- Business key from OLTP
    email VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    full_name VARCHAR(200) GENERATED ALWAYS AS (CONCAT(first_name, ' ', last_name)),
    date_of_birth DATE,
    age_group VARCHAR(20), -- Derived: '18-25', '26-35', etc.
    gender VARCHAR(10),
    city VARCHAR(100),
    state VARCHAR(50),
    country VARCHAR(50),
    customer_segment VARCHAR(20), -- 'Premium', 'Standard', 'Basic'
    
    -- SCD Type 2 fields
    effective_date DATE NOT NULL,
    expiry_date DATE NULL,
    is_current BOOLEAN NOT NULL DEFAULT TRUE,
    version_number INT NOT NULL DEFAULT 1,
    
    UNIQUE KEY uk_customer_version (customer_id, version_number),
    INDEX idx_customer_id (customer_id),
    INDEX idx_current (is_current),
    INDEX idx_segment (customer_segment)
);

-- Product Dimension
CREATE TABLE dim_product (
    product_key INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL, -- Business key
    product_name VARCHAR(200) NOT NULL,
    product_code VARCHAR(50),
    category_name VARCHAR(100) NOT NULL,
    subcategory_name VARCHAR(100),
    brand VARCHAR(100),
    color VARCHAR(50),
    size VARCHAR(50),
    weight DECIMAL(8,2),
    unit_cost DECIMAL(10,2),
    list_price DECIMAL(10,2),
    profit_margin DECIMAL(5,2) GENERATED ALWAYS AS 
        (CASE WHEN unit_cost > 0 THEN ((list_price - unit_cost) / unit_cost) * 100 ELSE 0 END),
    
    -- Product lifecycle
    launch_date DATE,
    discontinue_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    
    UNIQUE KEY uk_product_id (product_id),
    INDEX idx_category (category_name),
    INDEX idx_brand (brand),
    INDEX idx_active (is_active)
);

-- Sales Fact Table (Grain: One row per order line item)
CREATE TABLE fact_sales (
    sales_key BIGINT AUTO_INCREMENT PRIMARY KEY,
    
    -- Foreign keys to dimensions
    date_key INT NOT NULL,
    customer_key INT NOT NULL,
    product_key INT NOT NULL,
    
    -- Degenerate dimensions (stored in fact table)
    order_id INT NOT NULL,
    order_line_id INT NOT NULL,
    
    -- Measures (additive facts)
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL,
    cost_amount DECIMAL(10,2) NOT NULL,
    profit_amount DECIMAL(10,2) GENERATED ALWAYS AS (total_amount - cost_amount),
    
    -- Semi-additive measures
    inventory_on_hand INT,
    
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),
    FOREIGN KEY (product_key) REFERENCES dim_product(product_key),
    
    INDEX idx_date (date_key),
    INDEX idx_customer (customer_key),
    INDEX idx_product (product_key),
    INDEX idx_order (order_id),
    INDEX idx_composite (date_key, customer_key, product_key)
) PARTITION BY RANGE (date_key) (
    PARTITION p_2023 VALUES LESS THAN (20240101),
    PARTITION p_2024 VALUES LESS THAN (20250101),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 3.2 Additional Fact Tables

Create specialized fact tables for different analytical needs:

```sql
-- Customer Snapshot Fact (Monthly customer metrics)
CREATE TABLE fact_customer_snapshot (
    snapshot_key BIGINT AUTO_INCREMENT PRIMARY KEY,
    date_key INT NOT NULL,
    customer_key INT NOT NULL,
    
    -- Snapshot measures
    orders_count INT DEFAULT 0,
    total_revenue DECIMAL(12,2) DEFAULT 0,
    avg_order_value DECIMAL(10,2) DEFAULT 0,
    days_since_last_order INT,
    lifetime_orders INT DEFAULT 0,
    lifetime_revenue DECIMAL(12,2) DEFAULT 0,
    customer_tenure_days INT DEFAULT 0,
    
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),
    
    UNIQUE KEY uk_snapshot (date_key, customer_key),
    INDEX idx_date (date_key),
    INDEX idx_customer (customer_key)
);

-- Inventory Fact (Daily inventory snapshots)
CREATE TABLE fact_inventory (
    inventory_key BIGINT AUTO_INCREMENT PRIMARY KEY,
    date_key INT NOT NULL,
    product_key INT NOT NULL,
    
    -- Inventory measures
    beginning_inventory INT DEFAULT 0,
    ending_inventory INT DEFAULT 0,
    units_received INT DEFAULT 0,
    units_sold INT DEFAULT 0,
    units_adjusted INT DEFAULT 0,
    inventory_value DECIMAL(12,2) DEFAULT 0,
    days_of_supply INT,
    
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (product_key) REFERENCES dim_product(product_key),
    
    UNIQUE KEY uk_inventory (date_key, product_key),
    INDEX idx_date (date_key),
    INDEX idx_product (product_key)
);
```

## 4. Phase 2: ETL Process Implementation

### 4.1 Data Quality Framework

Implement data validation and cleansing:

```sql
-- Data Quality Rules Table
CREATE TABLE dq_rules (
    rule_id INT AUTO_INCREMENT PRIMARY KEY,
    rule_name VARCHAR(100) NOT NULL,
    table_name VARCHAR(64) NOT NULL,
    column_name VARCHAR(64),
    rule_type ENUM('NOT_NULL', 'RANGE', 'FORMAT', 'REFERENCE', 'CUSTOM') NOT NULL,
    rule_definition JSON NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Data Quality Results
CREATE TABLE dq_results (
    result_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rule_id INT NOT NULL,
    batch_id VARCHAR(50) NOT NULL,
    execution_date DATE NOT NULL,
    records_checked INT NOT NULL,
    records_failed INT NOT NULL,
    failure_rate DECIMAL(5,2) GENERATED ALWAYS AS ((records_failed / records_checked) * 100),
    sample_failures JSON,
    
    FOREIGN KEY (rule_id) REFERENCES dq_rules(rule_id),
    INDEX idx_batch (batch_id),
    INDEX idx_date (execution_date)
);

-- Data Quality Validation Procedure
DELIMITER //
CREATE PROCEDURE ValidateDataQuality(IN p_batch_id VARCHAR(50))
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_rule_id INT;
    DECLARE v_table_name VARCHAR(64);
    DECLARE v_rule_definition JSON;
    DECLARE v_records_checked INT DEFAULT 0;
    DECLARE v_records_failed INT DEFAULT 0;
    
    DECLARE rule_cursor CURSOR FOR 
        SELECT rule_id, table_name, rule_definition 
        FROM dq_rules 
        WHERE is_active = TRUE;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN rule_cursor;
    rule_loop: LOOP
        FETCH rule_cursor INTO v_rule_id, v_table_name, v_rule_definition;
        IF done THEN
            LEAVE rule_loop;
        END IF;
        
        -- Execute data quality checks based on rule type
        -- (Implementation varies by rule type)
        
        INSERT INTO dq_results (rule_id, batch_id, execution_date, records_checked, records_failed)
        VALUES (v_rule_id, p_batch_id, CURRENT_DATE, v_records_checked, v_records_failed);
        
    END LOOP;
    CLOSE rule_cursor;
END //
DELIMITER ;
```

### 4.2 Dimension Loading (SCD Type 2)

Implement slowly changing dimension logic:

```sql
-- Customer Dimension ETL Procedure
DELIMITER //
CREATE PROCEDURE LoadCustomerDimension()
BEGIN
    DECLARE v_batch_id VARCHAR(50);
    SET v_batch_id = CONCAT('CUST_', DATE_FORMAT(NOW(), '%Y%m%d_%H%i%s'));
    
    -- Expire changed records
    UPDATE dim_customer dc
    JOIN (
        SELECT 
            dc.customer_key,
            c.customer_id,
            c.first_name,
            c.last_name,
            c.city,
            c.state,
            c.country
        FROM techmart_db.customers c
        JOIN dim_customer dc ON c.customer_id = dc.customer_id
        WHERE dc.is_current = TRUE
          AND (dc.first_name != c.first_name 
               OR dc.last_name != c.last_name
               OR dc.city != COALESCE(c.city, '')
               OR dc.state != COALESCE(c.state, '')
               OR dc.country != COALESCE(c.country, ''))
    ) changes ON dc.customer_key = changes.customer_key
    SET dc.expiry_date = CURRENT_DATE - INTERVAL 1 DAY,
        dc.is_current = FALSE;
    
    -- Insert new and changed records
    INSERT INTO dim_customer (
        customer_id, email, first_name, last_name, date_of_birth,
        age_group, city, state, country, customer_segment,
        effective_date, version_number
    )
    SELECT 
        c.customer_id,
        c.email,
        c.first_name,
        c.last_name,
        c.date_of_birth,
        CASE 
            WHEN TIMESTAMPDIFF(YEAR, c.date_of_birth, CURRENT_DATE) BETWEEN 18 AND 25 THEN '18-25'
            WHEN TIMESTAMPDIFF(YEAR, c.date_of_birth, CURRENT_DATE) BETWEEN 26 AND 35 THEN '26-35'
            WHEN TIMESTAMPDIFF(YEAR, c.date_of_birth, CURRENT_DATE) BETWEEN 36 AND 50 THEN '36-50'
            WHEN TIMESTAMPDIFF(YEAR, c.date_of_birth, CURRENT_DATE) > 50 THEN '50+'
            ELSE 'Unknown'
        END as age_group,
        COALESCE(a.city, '') as city,
        COALESCE(a.state, '') as state,
        COALESCE(a.country, '') as country,
        CASE 
            WHEN customer_stats.lifetime_value > 5000 THEN 'Premium'
            WHEN customer_stats.lifetime_value > 1000 THEN 'Standard'
            ELSE 'Basic'
        END as customer_segment,
        CURRENT_DATE as effective_date,
        COALESCE(MAX(dc.version_number), 0) + 1 as version_number
    FROM techmart_db.customers c
    LEFT JOIN techmart_db.customer_addresses ca ON c.customer_id = ca.customer_id AND ca.is_primary = TRUE
    LEFT JOIN techmart_db.addresses a ON ca.address_id = a.address_id
    LEFT JOIN (
        SELECT 
            o.customer_id,
            SUM(oi.quantity * oi.unit_price) as lifetime_value
        FROM techmart_db.orders o
        JOIN techmart_db.order_items oi ON o.order_id = oi.order_id
        GROUP BY o.customer_id
    ) customer_stats ON c.customer_id = customer_stats.customer_id
    LEFT JOIN dim_customer dc ON c.customer_id = dc.customer_id
    WHERE NOT EXISTS (
        SELECT 1 FROM dim_customer dc2 
        WHERE dc2.customer_id = c.customer_id 
          AND dc2.is_current = TRUE
          AND dc2.first_name = c.first_name
          AND dc2.last_name = c.last_name
          AND dc2.city = COALESCE(a.city, '')
          AND dc2.state = COALESCE(a.state, '')
          AND dc2.country = COALESCE(a.country, '')
    )
    GROUP BY c.customer_id, c.email, c.first_name, c.last_name, c.date_of_birth,
             a.city, a.state, a.country, customer_stats.lifetime_value;
END //
DELIMITER ;
```

### 4.3 Fact Table Loading

Implement incremental fact loading:

```sql
-- Sales Fact ETL Procedure
DELIMITER //
CREATE PROCEDURE LoadSalesFact(IN p_load_date DATE)
BEGIN
    DECLARE v_batch_id VARCHAR(50);
    DECLARE v_date_key INT;
    
    SET v_batch_id = CONCAT('SALES_', DATE_FORMAT(p_load_date, '%Y%m%d'));
    SET v_date_key = CAST(DATE_FORMAT(p_load_date, '%Y%m%d') AS UNSIGNED);
    
    -- Delete existing data for the load date (idempotent loading)
    DELETE FROM fact_sales WHERE date_key = v_date_key;
    
    -- Load sales facts
    INSERT INTO fact_sales (
        date_key, customer_key, product_key, order_id, order_line_id,
        quantity, unit_price, discount_amount, tax_amount, total_amount, cost_amount
    )
    SELECT 
        v_date_key,
        dc.customer_key,
        dp.product_key,
        o.order_id,
        oi.order_item_id,
        oi.quantity,
        oi.unit_price,
        COALESCE(oi.discount_amount, 0),
        COALESCE(oi.tax_amount, 0),
        oi.quantity * oi.unit_price - COALESCE(oi.discount_amount, 0) + COALESCE(oi.tax_amount, 0),
        oi.quantity * p.unit_cost
    FROM techmart_db.orders o
    JOIN techmart_db.order_items oi ON o.order_id = oi.order_id
    JOIN techmart_db.products p ON oi.product_id = p.product_id
    JOIN dim_customer dc ON o.customer_id = dc.customer_id AND dc.is_current = TRUE
    JOIN dim_product dp ON oi.product_id = dp.product_id
    WHERE DATE(o.created_at) = p_load_date
      AND o.status IN ('completed', 'shipped', 'delivered');
    
    -- Log the batch
    INSERT INTO etl_batch_log (batch_id, table_name, load_date, records_processed, status)
    VALUES (v_batch_id, 'fact_sales', p_load_date, ROW_COUNT(), 'SUCCESS');
    
END //
DELIMITER ;
```

## 5. Phase 3: Complex Analytical Queries

### 5.1 Time Series Analysis

Create sophisticated analytical queries:

```sql
-- Monthly Revenue Trend with Year-over-Year Comparison
WITH monthly_sales AS (
    SELECT 
        dd.year,
        dd.month_number,
        dd.month_name,
        SUM(fs.total_amount) as monthly_revenue,
        COUNT(DISTINCT fs.order_id) as orders_count,
        COUNT(DISTINCT fs.customer_key) as unique_customers
    FROM fact_sales fs
    JOIN dim_date dd ON fs.date_key = dd.date_key
    WHERE dd.full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 24 MONTH)
    GROUP BY dd.year, dd.month_number, dd.month_name
),
yoy_comparison AS (
    SELECT 
        year,
        month_number,
        month_name,
        monthly_revenue,
        orders_count,
        unique_customers,
        LAG(monthly_revenue, 12) OVER (ORDER BY year, month_number) as prev_year_revenue,
        LAG(orders_count, 12) OVER (ORDER BY year, month_number) as prev_year_orders
    FROM monthly_sales
)
SELECT 
    year,
    month_name,
    monthly_revenue,
    prev_year_revenue,
    CASE 
        WHEN prev_year_revenue > 0 THEN 
            ROUND(((monthly_revenue - prev_year_revenue) / prev_year_revenue) * 100, 2)
        ELSE NULL 
    END as revenue_growth_pct,
    orders_count,
    prev_year_orders,
    CASE 
        WHEN prev_year_orders > 0 THEN 
            ROUND(((orders_count - prev_year_orders) / prev_year_orders) * 100, 2)
        ELSE NULL 
    END as orders_growth_pct,
    unique_customers,
    ROUND(monthly_revenue / orders_count, 2) as avg_order_value
FROM yoy_comparison
WHERE year = YEAR(CURRENT_DATE) OR year = YEAR(CURRENT_DATE) - 1
ORDER BY year, month_number;

-- Customer Cohort Analysis
WITH first_purchase AS (
    SELECT 
        fs.customer_key,
        MIN(dd.full_date) as first_purchase_date,
        DATE_FORMAT(MIN(dd.full_date), '%Y-%m') as cohort_month
    FROM fact_sales fs
    JOIN dim_date dd ON fs.date_key = dd.date_key
    GROUP BY fs.customer_key
),
customer_activities AS (
    SELECT 
        fp.customer_key,
        fp.cohort_month,
        dd.full_date as purchase_date,
        TIMESTAMPDIFF(MONTH, fp.first_purchase_date, dd.full_date) as months_since_first_purchase
    FROM first_purchase fp
    JOIN fact_sales fs ON fp.customer_key = fs.customer_key
    JOIN dim_date dd ON fs.date_key = dd.date_key
),
cohort_data AS (
    SELECT 
        cohort_month,
        months_since_first_purchase,
        COUNT(DISTINCT customer_key) as customers
    FROM customer_activities
    GROUP BY cohort_month, months_since_first_purchase
),
cohort_sizes AS (
    SELECT 
        cohort_month,
        COUNT(DISTINCT customer_key) as cohort_size
    FROM first_purchase
    GROUP BY cohort_month
)
SELECT 
    cd.cohort_month,
    cs.cohort_size,
    cd.months_since_first_purchase,
    cd.customers,
    ROUND((cd.customers * 100.0) / cs.cohort_size, 2) as retention_rate
FROM cohort_data cd
JOIN cohort_sizes cs ON cd.cohort_month = cs.cohort_month
WHERE cd.cohort_month >= DATE_FORMAT(DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH), '%Y-%m')
ORDER BY cd.cohort_month, cd.months_since_first_purchase;
```

### 5.2 Advanced Product Analytics

```sql
-- Product Performance Analysis with ABC Classification
WITH product_metrics AS (
    SELECT 
        dp.product_key,
        dp.product_name,
        dp.category_name,
        dp.brand,
        SUM(fs.quantity) as total_quantity_sold,
        SUM(fs.total_amount) as total_revenue,
        SUM(fs.profit_amount) as total_profit,
        COUNT(DISTINCT fs.customer_key) as unique_customers,
        AVG(fs.unit_price) as avg_selling_price,
        STDDEV(fs.unit_price) as price_volatility
    FROM fact_sales fs
    JOIN dim_product dp ON fs.product_key = dp.product_key
    JOIN dim_date dd ON fs.date_key = dd.date_key
    WHERE dd.full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
    GROUP BY dp.product_key, dp.product_name, dp.category_name, dp.brand
),
revenue_percentiles AS (
    SELECT 
        *,
        PERCENT_RANK() OVER (ORDER BY total_revenue DESC) as revenue_percentile,
        ROW_NUMBER() OVER (ORDER BY total_revenue DESC) as revenue_rank
    FROM product_metrics
),
abc_classification AS (
    SELECT 
        *,
        CASE 
            WHEN revenue_percentile <= 0.2 THEN 'A'
            WHEN revenue_percentile <= 0.5 THEN 'B'
            ELSE 'C'
        END as abc_class
    FROM revenue_percentiles
)
SELECT 
    product_name,
    category_name,
    brand,
    abc_class,
    total_revenue,
    total_profit,
    ROUND((total_profit / total_revenue) * 100, 2) as profit_margin_pct,
    total_quantity_sold,
    unique_customers,
    ROUND(total_revenue / unique_customers, 2) as revenue_per_customer,
    revenue_rank
FROM abc_classification
ORDER BY total_revenue DESC;

-- Inventory Turnover Analysis
SELECT 
    dp.category_name,
    dp.product_name,
    AVG(fi.beginning_inventory + fi.ending_inventory) / 2 as avg_inventory,
    SUM(fs.quantity) as units_sold,
    CASE 
        WHEN AVG(fi.beginning_inventory + fi.ending_inventory) > 0 THEN
            SUM(fs.quantity) / (AVG(fi.beginning_inventory + fi.ending_inventory) / 2)
        ELSE 0 
    END as inventory_turnover_ratio,
    CASE 
        WHEN SUM(fs.quantity) > 0 THEN
            365 / (SUM(fs.quantity) / (AVG(fi.beginning_inventory + fi.ending_inventory) / 2))
        ELSE NULL 
    END as days_in_inventory
FROM dim_product dp
LEFT JOIN fact_inventory fi ON dp.product_key = fi.product_key
LEFT JOIN fact_sales fs ON dp.product_key = fs.product_key
JOIN dim_date dd ON fs.date_key = dd.date_key
WHERE dd.full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY dp.category_name, dp.product_name
HAVING avg_inventory > 0
ORDER BY inventory_turnover_ratio DESC;
```

## 6. Phase 4: Automated Reporting

### 6.1 Executive Dashboard Queries

Create pre-aggregated views for dashboards:

```sql
-- Executive Summary View
CREATE VIEW v_executive_summary AS
WITH daily_metrics AS (
    SELECT 
        dd.full_date,
        SUM(fs.total_amount) as daily_revenue,
        COUNT(DISTINCT fs.order_id) as daily_orders,
        COUNT(DISTINCT fs.customer_key) as daily_customers,
        AVG(fs.total_amount) as avg_order_value
    FROM fact_sales fs
    JOIN dim_date dd ON fs.date_key = dd.date_key
    WHERE dd.full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
    GROUP BY dd.full_date
)
SELECT 
    'Last 30 Days' as period,
    SUM(daily_revenue) as total_revenue,
    AVG(daily_revenue) as avg_daily_revenue,
    SUM(daily_orders) as total_orders,
    AVG(daily_orders) as avg_daily_orders,
    SUM(daily_customers) as unique_customers,
    AVG(avg_order_value) as avg_order_value,
    
    -- Week-over-week comparison
    (SELECT SUM(daily_revenue) FROM daily_metrics WHERE full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 7 DAY)) as last_week_revenue,
    (SELECT SUM(daily_revenue) FROM daily_metrics WHERE full_date BETWEEN DATE_SUB(CURRENT_DATE, INTERVAL 14 DAY) AND DATE_SUB(CURRENT_DATE, INTERVAL 8 DAY)) as prev_week_revenue
FROM daily_metrics;

-- Category Performance View
CREATE VIEW v_category_performance AS
SELECT 
    dp.category_name,
    COUNT(DISTINCT dp.product_key) as products_count,
    SUM(fs.total_amount) as category_revenue,
    SUM(fs.quantity) as units_sold,
    AVG(fs.unit_price) as avg_price,
    SUM(fs.profit_amount) as category_profit,
    ROUND((SUM(fs.profit_amount) / SUM(fs.total_amount)) * 100, 2) as profit_margin_pct,
    
    -- Revenue share
    ROUND((SUM(fs.total_amount) / (SELECT SUM(total_amount) FROM fact_sales fs2 JOIN dim_date dd2 ON fs2.date_key = dd2.date_key WHERE dd2.full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH))) * 100, 2) as revenue_share_pct
FROM fact_sales fs
JOIN dim_product dp ON fs.product_key = dp.product_key
JOIN dim_date dd ON fs.date_key = dd.date_key
WHERE dd.full_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY dp.category_name
ORDER BY category_revenue DESC;
```

### 6.2 Automated Report Generation

```sql
-- Automated Daily Report Procedure
DELIMITER //
CREATE PROCEDURE GenerateDailyReport(IN p_report_date DATE)
BEGIN
    DECLARE v_report_id INT;
    DECLARE v_total_revenue DECIMAL(12,2);
    DECLARE v_total_orders INT;
    DECLARE v_unique_customers INT;
    
    -- Insert report header
    INSERT INTO report_executions (report_name, report_date, status, started_at)
    VALUES ('Daily Sales Report', p_report_date, 'RUNNING', NOW());
    
    SET v_report_id = LAST_INSERT_ID();
    
    -- Calculate key metrics
    SELECT 
        COALESCE(SUM(fs.total_amount), 0),
        COUNT(DISTINCT fs.order_id),
        COUNT(DISTINCT fs.customer_key)
    INTO v_total_revenue, v_total_orders, v_unique_customers
    FROM fact_sales fs
    JOIN dim_date dd ON fs.date_key = dd.date_key
    WHERE dd.full_date = p_report_date;
    
    -- Store report data
    INSERT INTO daily_report_data (
        report_id, report_date, total_revenue, total_orders, 
        unique_customers, avg_order_value
    ) VALUES (
        v_report_id, p_report_date, v_total_revenue, v_total_orders,
        v_unique_customers, CASE WHEN v_total_orders > 0 THEN v_total_revenue / v_total_orders ELSE 0 END
    );
    
    -- Update report status
    UPDATE report_executions 
    SET status = 'COMPLETED', completed_at = NOW()
    WHERE report_id = v_report_id;
    
END //
DELIMITER ;

-- Schedule daily report (would be called by cron job)
-- 0 1 * * * mysql -u etl_user -p techmart_dw -e "CALL GenerateDailyReport(CURRENT_DATE - INTERVAL 1 DAY);"
```

## 7. Phase 5: Performance Optimization

### 7.1 Aggregation Tables

Create pre-aggregated summary tables:

```sql
-- Monthly Sales Aggregates
CREATE TABLE agg_monthly_sales (
    agg_key INT AUTO_INCREMENT PRIMARY KEY,
    year SMALLINT NOT NULL,
    month_number TINYINT NOT NULL,
    category_name VARCHAR(100) NOT NULL,
    total_revenue DECIMAL(15,2) NOT NULL,
    total_orders INT NOT NULL,
    unique_customers INT NOT NULL,
    avg_order_value DECIMAL(10,2) NOT NULL,
    units_sold INT NOT NULL,
    
    UNIQUE KEY uk_month_category (year, month_number, category_name),
    INDEX idx_year_month (year, month_number)
);

-- Populate monthly aggregates
INSERT INTO agg_monthly_sales (
    year, month_number, category_name, total_revenue, 
    total_orders, unique_customers, avg_order_value, units_sold
)
SELECT 
    dd.year,
    dd.month_number,
    dp.category_name,
    SUM(fs.total_amount) as total_revenue,
    COUNT(DISTINCT fs.order_id) as total_orders,
    COUNT(DISTINCT fs.customer_key) as unique_customers,
    AVG(fs.total_amount) as avg_order_value,
    SUM(fs.quantity) as units_sold
FROM fact_sales fs
JOIN dim_date dd ON fs.date_key = dd.date_key
JOIN dim_product dp ON fs.product_key = dp.product_key
GROUP BY dd.year, dd.month_number, dp.category_name
ON DUPLICATE KEY UPDATE
    total_revenue = VALUES(total_revenue),
    total_orders = VALUES(total_orders),
    unique_customers = VALUES(unique_customers),
    avg_order_value = VALUES(avg_order_value),
    units_sold = VALUES(units_sold);
```

### 7.2 Columnstore Indexes (MySQL 8.0+)

```sql
-- Create columnstore index for analytical queries
-- Note: This is a MySQL 8.0+ feature
ALTER TABLE fact_sales ADD INDEX idx_columnstore (
    date_key, customer_key, product_key, total_amount, quantity
) USING COLUMNSTORE;

-- Optimize for analytical workloads
SET SESSION optimizer_switch = 'hash_join=on';
SET SESSION join_buffer_size = 256M;
```

## 8. Phase 6: Data Quality and Monitoring

### 8.1 Data Quality Monitoring

```sql
-- Data Quality Dashboard
CREATE VIEW v_data_quality_summary AS
SELECT 
    DATE(execution_date) as check_date,
    COUNT(*) as total_checks,
    SUM(CASE WHEN failure_rate = 0 THEN 1 ELSE 0 END) as passed_checks,
    SUM(CASE WHEN failure_rate > 0 THEN 1 ELSE 0 END) as failed_checks,
    AVG(failure_rate) as avg_failure_rate,
    MAX(failure_rate) as max_failure_rate
FROM dq_results
WHERE execution_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY DATE(execution_date)
ORDER BY check_date DESC;

-- Data Freshness Monitoring
CREATE VIEW v_data_freshness AS
SELECT 
    'fact_sales' as table_name,
    MAX(dd.full_date) as latest_data_date,
    DATEDIFF(CURRENT_DATE, MAX(dd.full_date)) as days_behind,
    CASE 
        WHEN DATEDIFF(CURRENT_DATE, MAX(dd.full_date)) <= 1 THEN 'Fresh'
        WHEN DATEDIFF(CURRENT_DATE, MAX(dd.full_date)) <= 3 THEN 'Stale'
        ELSE 'Critical'
    END as freshness_status
FROM fact_sales fs
JOIN dim_date dd ON fs.date_key = dd.date_key

UNION ALL

SELECT 
    'dim_customer' as table_name,
    MAX(effective_date) as latest_data_date,
    DATEDIFF(CURRENT_DATE, MAX(effective_date)) as days_behind,
    CASE 
        WHEN DATEDIFF(CURRENT_DATE, MAX(effective_date)) <= 1 THEN 'Fresh'
        WHEN DATEDIFF(CURRENT_DATE, MAX(effective_date)) <= 7 THEN 'Stale'
        ELSE 'Critical'
    END as freshness_status
FROM dim_customer
WHERE is_current = TRUE;
```

### 8.2 Performance Monitoring

```sql
-- Query Performance Monitoring
CREATE TABLE query_performance_log (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    query_name VARCHAR(100) NOT NULL,
    execution_time_ms INT NOT NULL,
    rows_examined BIGINT,
    rows_returned INT,
    execution_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_query_date (query_name, execution_date),
    INDEX idx_execution_time (execution_time_ms)
);

-- ETL Performance Monitoring
CREATE VIEW v_etl_performance AS
SELECT 
    table_name,
    DATE(load_date) as load_date,
    AVG(records_processed) as avg_records,
    AVG(TIMESTAMPDIFF(SECOND, started_at, completed_at)) as avg_duration_seconds,
    COUNT(*) as execution_count,
    SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) as successful_runs,
    SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) as failed_runs
FROM etl_batch_log
WHERE load_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY table_name, DATE(load_date)
ORDER BY load_date DESC, table_name;
```

## 9. Deliverables and Assessment

### 9.1 Required Deliverables

Submit the following:

1. **Dimensional Model Design** (Star schema ERD and documentation)
2. **Complete DDL Scripts** (All table creation statements)
3. **ETL Procedures** (Data loading and transformation logic)
4. **Data Quality Framework** (Validation rules and monitoring)
5. **Analytical Query Library** (At least 20 complex analytical queries)
6. **Performance Optimization** (Indexes, partitioning, aggregation tables)
7. **Automated Reporting** (Dashboard views and report procedures)
8. **Monitoring Dashboard** (Data quality and performance metrics)

### 9.2 Evaluation Criteria

**Dimensional Modeling (30%)**
- Proper star schema design
- Appropriate grain selection
- SCD implementation
- Business key management

**ETL Implementation (25%)**
- Data quality and validation
- Incremental loading strategies
- Error handling and logging
- Performance optimization

**Analytical Capabilities (25%)**
- Complex query implementation
- Advanced SQL techniques
- Business intelligence insights
- Query performance

**Operational Excellence (20%)**
- Monitoring and alerting
- Automated processes
- Documentation quality
- Scalability considerations

## 10. Extension Challenges

### 10.1 Advanced Analytics
- Implement predictive models using SQL
- Create customer churn prediction queries
- Build recommendation engine logic
- Implement statistical analysis functions

### 10.2 Real-time Analytics
- Design near real-time data pipeline
- Implement change data capture (CDC)
- Create streaming aggregations
- Build operational dashboards

### 10.3 Advanced Optimization
- Implement columnar storage strategies
- Design distributed query processing
- Create intelligent data archiving
- Build automated performance tuning

---

**Navigation**

[← Previous: Project 1: E-commerce Database Design](project-01-ecommerce-database.md) | [Next → Project 3: High-Availability System](project-03-high-availability.md)

_Last updated: 2025-01-21_ 