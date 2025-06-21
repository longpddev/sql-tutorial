# Project 2: Data Warehouse Implementation

> **Assessment Project • Analytics Focus**  
> Estimated time: 6-8 hours | Difficulty: ★★★★☆

## 1. Project Overview

Build a comprehensive data warehouse for business intelligence using star schema design. Transform operational data into analytical insights with ETL processes, dimensional modeling, and performance-optimized reporting.

**Learning Objectives:**
- Design star schema for analytics
- Implement ETL pipelines
- Create OLAP cubes and aggregations
- Build performance-optimized reports
- Handle slowly changing dimensions

> **Prerequisites:** Complete [Data Modeling for Different Workloads](10-01-data-modeling-workloads.md) and [Schema Design Patterns](05-01-schema-design-patterns.md).

## 2. Business Requirements

### 2.1 Analytics Needs
- Sales performance analysis by time, product, customer, geography
- Customer behavior and segmentation analysis
- Inventory turnover and profitability reports
- Trend analysis and forecasting data preparation
- Executive dashboards with KPIs

### 2.2 Data Sources
- Operational e-commerce database (from Project 1)
- External customer demographics data
- Marketing campaign data
- Website analytics data

## 3. Data Warehouse Schema

### 3.1 Dimensional Model Design

```sql
-- Create data warehouse database
CREATE DATABASE ecommerce_dw;
USE ecommerce_dw;

-- Date dimension (comprehensive)
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE NOT NULL,
    day_of_week TINYINT,
    day_name VARCHAR(10),
    day_of_month TINYINT,
    day_of_year SMALLINT,
    week_of_year TINYINT,
    month_number TINYINT,
    month_name VARCHAR(10),
    quarter TINYINT,
    year SMALLINT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    fiscal_year SMALLINT,
    fiscal_quarter TINYINT
);

-- Customer dimension with SCD Type 2
CREATE TABLE dim_customer (
    customer_key INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    email VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    full_name VARCHAR(200),
    customer_group VARCHAR(50),
    customer_segment VARCHAR(50),
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(50),
    effective_date DATE NOT NULL,
    expiry_date DATE,
    is_current BOOLEAN DEFAULT TRUE,
    INDEX idx_customer_id (customer_id),
    INDEX idx_current (is_current)
);

-- Product dimension
CREATE TABLE dim_product (
    product_key INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    sku VARCHAR(50),
    product_name VARCHAR(200),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    cost_price DECIMAL(10,2),
    effective_date DATE NOT NULL,
    expiry_date DATE,
    is_current BOOLEAN DEFAULT TRUE
);

-- Sales fact table
CREATE TABLE fact_sales (
    sales_key BIGINT AUTO_INCREMENT PRIMARY KEY,
    date_key INT NOT NULL,
    customer_key INT NOT NULL,
    product_key INT NOT NULL,
    order_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_sales DECIMAL(12,2) NOT NULL,
    cost_of_goods DECIMAL(12,2),
    gross_profit DECIMAL(12,2),
    discount_amount DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),
    FOREIGN KEY (product_key) REFERENCES dim_product(product_key),
    INDEX idx_date (date_key),
    INDEX idx_customer (customer_key),
    INDEX idx_product (product_key)
) PARTITION BY RANGE (date_key) (
    PARTITION p2023 VALUES LESS THAN (20240101),
    PARTITION p2024 VALUES LESS THAN (20250101),
    PARTITION p2025 VALUES LESS THAN (20260101)
);
```

### 3.2 ETL Implementation

```sql
-- ETL procedures for data warehouse loading

DELIMITER //

-- Load date dimension
CREATE PROCEDURE LoadDateDimension(IN start_date DATE, IN end_date DATE)
BEGIN
    DECLARE current_date DATE DEFAULT start_date;
    
    WHILE current_date <= end_date DO
        INSERT IGNORE INTO dim_date (
            date_key, full_date, day_of_week, day_name, day_of_month,
            day_of_year, week_of_year, month_number, month_name,
            quarter, year, is_weekend, is_holiday, fiscal_year, fiscal_quarter
        ) VALUES (
            DATE_FORMAT(current_date, '%Y%m%d'),
            current_date,
            DAYOFWEEK(current_date),
            DAYNAME(current_date),
            DAY(current_date),
            DAYOFYEAR(current_date),
            WEEK(current_date),
            MONTH(current_date),
            MONTHNAME(current_date),
            QUARTER(current_date),
            YEAR(current_date),
            DAYOFWEEK(current_date) IN (1, 7),
            FALSE, -- Holiday logic would go here
            CASE WHEN MONTH(current_date) >= 4 THEN YEAR(current_date) ELSE YEAR(current_date) - 1 END,
            CASE WHEN MONTH(current_date) >= 4 THEN QUARTER(current_date) ELSE QUARTER(current_date) + 4 END
        );
        
        SET current_date = DATE_ADD(current_date, INTERVAL 1 DAY);
    END WHILE;
END //

-- Load customer dimension with SCD Type 2
CREATE PROCEDURE LoadCustomerDimension()
BEGIN
    -- Insert new customers
    INSERT INTO dim_customer (
        customer_id, email, first_name, last_name, full_name,
        customer_group, customer_segment, city, state, country, effective_date
    )
    SELECT 
        c.id,
        c.email,
        c.first_name,
        c.last_name,
        CONCAT(c.first_name, ' ', c.last_name),
        c.customer_group,
        CASE 
            WHEN ca.total_orders >= 10 THEN 'VIP'
            WHEN ca.total_orders >= 5 THEN 'Regular'
            ELSE 'New'
        END,
        a.city,
        a.state,
        a.country,
        CURDATE()
    FROM ecommerce_system.customers c
    LEFT JOIN ecommerce_system.customer_addresses a ON c.id = a.customer_id AND a.is_default = TRUE
    LEFT JOIN ecommerce_system.customer_analytics ca ON c.id = ca.customer_id
    WHERE c.id NOT IN (SELECT customer_id FROM dim_customer WHERE is_current = TRUE);
    
    -- Handle changes (SCD Type 2)
    -- This is simplified - in practice, you'd detect specific changes
    UPDATE dim_customer dc
    JOIN ecommerce_system.customers c ON dc.customer_id = c.id
    SET dc.expiry_date = DATE_SUB(CURDATE(), INTERVAL 1 DAY),
        dc.is_current = FALSE
    WHERE dc.is_current = TRUE
      AND (dc.email != c.email OR dc.customer_group != c.customer_group);
END //

DELIMITER ;
```

## 4. Implementation Tasks

### Task 1: Dimensional Design (1.5 hours)
- Create all dimension and fact tables
- Implement proper indexing strategy
- Add data quality constraints

### Task 2: ETL Development (2.5 hours)
- Build ETL procedures for each dimension
- Implement incremental loading
- Add error handling and logging

### Task 3: Aggregation Tables (1 hour)
- Create summary tables for performance
- Implement automatic refresh procedures

### Task 4: Analytics Views (1.5 hours)
- Build business-friendly reporting views
- Create KPI calculation views

### Task 5: Performance Testing (1 hour)
- Test query performance on large datasets
- Optimize slow-running queries

## 5. Sample Analytics Queries

```sql
-- Monthly sales trend
SELECT 
    d.year,
    d.month_name,
    SUM(f.total_sales) as monthly_sales,
    COUNT(DISTINCT f.customer_key) as unique_customers,
    AVG(f.total_sales) as avg_order_value
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year = 2024
GROUP BY d.year, d.month_number, d.month_name
ORDER BY d.month_number;

-- Top products by profitability
SELECT 
    p.product_name,
    p.category,
    SUM(f.quantity) as units_sold,
    SUM(f.total_sales) as revenue,
    SUM(f.gross_profit) as profit,
    AVG(f.gross_profit / f.total_sales) * 100 as profit_margin
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year = 2024
GROUP BY p.product_key, p.product_name, p.category
ORDER BY profit DESC
LIMIT 10;

-- Customer segmentation analysis
SELECT 
    c.customer_segment,
    COUNT(DISTINCT c.customer_key) as customer_count,
    SUM(f.total_sales) as segment_revenue,
    AVG(f.total_sales) as avg_order_value,
    SUM(f.quantity) as total_items
FROM fact_sales f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year = 2024 AND c.is_current = TRUE
GROUP BY c.customer_segment
ORDER BY segment_revenue DESC;
```

## 6. Deliverables

1. **Complete dimensional model** with star schema
2. **ETL procedures** for all data sources
3. **Performance-optimized** aggregation tables
4. **Analytics views** for business reporting
5. **Sample dashboard queries** with results
6. **Performance analysis** and optimization report

## 7. Evaluation Criteria

- **Schema Design (30%)**: Proper dimensional modeling, SCD implementation
- **ETL Implementation (25%)**: Complete, efficient data loading processes
- **Performance (20%)**: Query optimization, appropriate indexing
- **Analytics (15%)**: Meaningful business insights and KPIs
- **Documentation (10%)**: Clear explanation of design decisions

---

**Navigation**

[← Previous: E-commerce Database Design Project](project-01-ecommerce-database.md) | [Next → High-Availability System Project](project-03-high-availability.md)

_Last updated: 2025-01-21_ 