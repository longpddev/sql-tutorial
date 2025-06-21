# Project 1: E-commerce Database Design

> **Capstone Project • Assessment 1**  
> Estimated time: 4-6 hours | Difficulty: ★★★★☆

## 1. Project Overview

Design and implement a complete e-commerce database system from scratch. This project integrates concepts from all previous modules, requiring you to make real-world design decisions, implement complex business logic, and optimize for performance. You'll create a production-ready schema that could power an actual online store.

**Learning Objectives:**
- Apply normalization principles to complex business requirements
- Implement referential integrity and business constraints
- Design for both OLTP operations and analytical reporting
- Handle concurrent user scenarios and data consistency
- Optimize queries for performance at scale

## 2. Business Requirements

You're building the database for "TechMart" - an online electronics retailer with these requirements:

### Core Entities
- **Customers**: Registration, profiles, addresses, payment methods
- **Products**: Catalog with categories, variants, inventory tracking
- **Orders**: Shopping cart, checkout, order fulfillment
- **Payments**: Multiple payment methods, transaction tracking
- **Shipping**: Address management, shipping methods, tracking
- **Reviews**: Product ratings and reviews system

### Business Rules
1. Customers can have multiple shipping addresses
2. Products can have multiple variants (size, color, etc.)
3. Inventory must be tracked per product variant
4. Orders can contain multiple products with quantities
5. Payment can be split across multiple methods
6. Order status must be tracked through fulfillment pipeline
7. Reviews can only be left by customers who purchased the product
8. Pricing can vary by customer segment or promotional periods

### Performance Requirements
- Support 10,000+ concurrent users
- Handle 1M+ products with variants
- Process 50,000+ orders per day
- Generate real-time inventory reports
- Support complex product search and filtering

## 3. Phase 1: Schema Design

### 3.1 Entity Relationship Design

Design your schema following these steps:

```sql
-- Start with core entities and relationships
-- Focus on proper normalization (3NF minimum)

-- Example starting structure (expand this):
CREATE TABLE customers (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    
    INDEX idx_email (email),
    INDEX idx_created_at (created_at)
);

-- TODO: Design the complete schema
-- Include all entities: products, categories, orders, payments, etc.
```

**Design Tasks:**
1. Create a complete ERD (Entity Relationship Diagram)
2. Identify all entities, attributes, and relationships
3. Apply normalization rules (eliminate redundancy)
4. Define primary keys, foreign keys, and constraints
5. Plan indexes for performance

### 3.2 Advanced Schema Features

Implement these advanced patterns:

**Soft Deletes Pattern:**
```sql
-- Instead of DELETE, use status flags
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMP NULL;
ALTER TABLE customers ADD COLUMN is_active BOOLEAN DEFAULT TRUE;

-- Create partial indexes for active records only
CREATE INDEX idx_active_products ON products (category_id) 
WHERE deleted_at IS NULL;
```

**Audit Trail Pattern:**
```sql
-- Track all changes for compliance
CREATE TABLE audit_log (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(64) NOT NULL,
    record_id INT NOT NULL,
    operation ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    old_values JSON,
    new_values JSON,
    changed_by INT NOT NULL,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_table_record (table_name, record_id),
    INDEX idx_changed_at (changed_at)
);
```

**Temporal Data Pattern:**
```sql
-- Track price history
CREATE TABLE product_price_history (
    price_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP NULL,
    created_by INT NOT NULL,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    INDEX idx_product_validity (product_id, valid_from, valid_to)
);
```

## 4. Phase 2: Implementation and Data Loading

### 4.1 Create Complete Schema

Implement your full schema with:

```sql
-- Set up proper SQL mode for strict validation
SET sql_mode = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';

-- Create database with proper charset
CREATE DATABASE techmart_db 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

USE techmart_db;

-- Implement all your tables here
-- Remember: Create tables in dependency order (referenced tables first)

-- Example: Categories before Products
CREATE TABLE categories (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    parent_category_id INT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id),
    INDEX idx_parent (parent_category_id),
    INDEX idx_active (is_active)
);
```

### 4.2 Load Sample Data

Create realistic test data:

```sql
-- Use stored procedures or scripts to generate large datasets
DELIMITER //
CREATE PROCEDURE GenerateCustomers(IN num_customers INT)
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= num_customers DO
        INSERT INTO customers (email, password_hash, first_name, last_name, phone)
        VALUES (
            CONCAT('user', i, '@techmart.com'),
            SHA2(CONCAT('password', i), 256),
            CONCAT('FirstName', i),
            CONCAT('LastName', i),
            CONCAT('555-', LPAD(i, 7, '0'))
        );
        SET i = i + 1;
    END WHILE;
END //
DELIMITER ;

-- Generate test data
CALL GenerateCustomers(10000);
-- Create similar procedures for products, orders, etc.
```

**Data Requirements:**
- 10,000+ customers
- 1,000+ products across multiple categories
- 50,000+ orders with realistic patterns
- Product reviews and ratings
- Multiple addresses per customer
- Inventory transactions

## 5. Phase 3: Business Logic Implementation

### 5.1 Core CRUD Operations

Implement essential business operations:

```sql
-- Customer Registration
DELIMITER //
CREATE PROCEDURE RegisterCustomer(
    IN p_email VARCHAR(255),
    IN p_password VARCHAR(255),
    IN p_first_name VARCHAR(100),
    IN p_last_name VARCHAR(100),
    OUT p_customer_id INT,
    OUT p_result VARCHAR(100)
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_result = 'Registration failed';
        SET p_customer_id = 0;
    END;
    
    START TRANSACTION;
    
    -- Check if email already exists
    IF EXISTS (SELECT 1 FROM customers WHERE email = p_email) THEN
        SET p_result = 'Email already registered';
        SET p_customer_id = 0;
        ROLLBACK;
    ELSE
        INSERT INTO customers (email, password_hash, first_name, last_name)
        VALUES (p_email, SHA2(p_password, 256), p_first_name, p_last_name);
        
        SET p_customer_id = LAST_INSERT_ID();
        SET p_result = 'Success';
        COMMIT;
    END IF;
END //
DELIMITER ;
```

**Required Operations:**
1. Customer registration and authentication
2. Product catalog browsing with search/filtering
3. Shopping cart management
4. Order placement with inventory checking
5. Payment processing simulation
6. Order status tracking
7. Review and rating system

### 5.2 Complex Business Logic

Implement advanced scenarios:

```sql
-- Order Placement with Inventory Check
DELIMITER //
CREATE PROCEDURE PlaceOrder(
    IN p_customer_id INT,
    IN p_items JSON, -- [{"product_id": 1, "quantity": 2}, ...]
    OUT p_order_id INT,
    OUT p_result VARCHAR(100)
)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_product_id INT;
    DECLARE v_quantity INT;
    DECLARE v_available_stock INT;
    DECLARE v_price DECIMAL(10,2);
    DECLARE v_total DECIMAL(10,2) DEFAULT 0;
    
    DECLARE item_cursor CURSOR FOR 
        SELECT JSON_UNQUOTE(JSON_EXTRACT(value, '$.product_id')),
               JSON_UNQUOTE(JSON_EXTRACT(value, '$.quantity'))
        FROM JSON_TABLE(p_items, '$[*]' COLUMNS (value JSON PATH '$')) AS jt;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_result = 'Order placement failed';
        SET p_order_id = 0;
    END;
    
    START TRANSACTION;
    
    -- Validate all items have sufficient stock first
    OPEN item_cursor;
    check_loop: LOOP
        FETCH item_cursor INTO v_product_id, v_quantity;
        IF done THEN
            LEAVE check_loop;
        END IF;
        
        SELECT stock_quantity, price INTO v_available_stock, v_price
        FROM products 
        WHERE product_id = v_product_id AND is_active = TRUE;
        
        IF v_available_stock < v_quantity THEN
            SET p_result = CONCAT('Insufficient stock for product ', v_product_id);
            SET p_order_id = 0;
            ROLLBACK;
            LEAVE check_loop;
        END IF;
        
        SET v_total = v_total + (v_price * v_quantity);
    END LOOP;
    CLOSE item_cursor;
    
    -- If we get here, all items are available
    -- Create order and deduct inventory
    -- (Implementation continues...)
    
    COMMIT;
END //
DELIMITER ;
```

## 6. Phase 4: Reporting and Analytics

### 6.1 Business Intelligence Queries

Create reports that business stakeholders need:

```sql
-- Daily Sales Report
SELECT 
    DATE(o.created_at) as order_date,
    COUNT(DISTINCT o.order_id) as total_orders,
    COUNT(DISTINCT o.customer_id) as unique_customers,
    SUM(oi.quantity * oi.unit_price) as total_revenue,
    AVG(oi.quantity * oi.unit_price) as avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.created_at >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
GROUP BY DATE(o.created_at)
ORDER BY order_date DESC;

-- Top Selling Products
SELECT 
    p.name,
    c.name as category,
    SUM(oi.quantity) as total_sold,
    SUM(oi.quantity * oi.unit_price) as total_revenue,
    AVG(r.rating) as avg_rating,
    COUNT(r.review_id) as review_count
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN reviews r ON p.product_id = r.product_id
WHERE p.is_active = TRUE
GROUP BY p.product_id, p.name, c.name
ORDER BY total_sold DESC
LIMIT 20;

-- Customer Lifetime Value
WITH customer_metrics AS (
    SELECT 
        c.customer_id,
        c.first_name,
        c.last_name,
        COUNT(DISTINCT o.order_id) as total_orders,
        SUM(oi.quantity * oi.unit_price) as lifetime_value,
        AVG(oi.quantity * oi.unit_price) as avg_order_value,
        DATEDIFF(CURRENT_DATE, MAX(o.created_at)) as days_since_last_order
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    WHERE c.is_active = TRUE
    GROUP BY c.customer_id, c.first_name, c.last_name
)
SELECT 
    customer_id,
    CONCAT(first_name, ' ', last_name) as customer_name,
    total_orders,
    lifetime_value,
    avg_order_value,
    CASE 
        WHEN days_since_last_order <= 30 THEN 'Active'
        WHEN days_since_last_order <= 90 THEN 'At Risk'
        ELSE 'Inactive'
    END as customer_status
FROM customer_metrics
WHERE lifetime_value > 0
ORDER BY lifetime_value DESC;
```

### 6.2 Performance Analytics

Monitor system performance:

```sql
-- Slow Query Analysis
SELECT 
    table_name,
    COUNT(*) as query_count,
    AVG(query_time) as avg_query_time,
    MAX(query_time) as max_query_time
FROM performance_schema.events_statements_history_long
WHERE sql_text LIKE '%techmart_db%'
GROUP BY table_name
ORDER BY avg_query_time DESC;

-- Index Usage Analysis
SELECT 
    table_name,
    index_name,
    cardinality,
    CASE 
        WHEN cardinality = 0 THEN 'Unused'
        WHEN cardinality < 10 THEN 'Low Selectivity'
        ELSE 'Good'
    END as index_status
FROM information_schema.statistics
WHERE table_schema = 'techmart_db'
ORDER BY table_name, cardinality DESC;
```

## 7. Phase 5: Performance Optimization

### 7.1 Query Optimization

Identify and fix performance bottlenecks:

```sql
-- Before optimization: Slow product search
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE p.name LIKE '%laptop%'
  AND p.price BETWEEN 500 AND 2000
  AND p.is_active = TRUE
ORDER BY p.created_at DESC;

-- Add proper indexes
CREATE INDEX idx_product_search ON products (name, price, is_active, created_at);
CREATE INDEX idx_product_category_active ON products (category_id, is_active);

-- Optimized query with covering index
CREATE INDEX idx_product_covering ON products (
    is_active, price, name, product_id, category_id, created_at
);
```

### 7.2 Scaling Considerations

Implement patterns for high-traffic scenarios:

```sql
-- Read Replica Configuration
-- Configure read-only queries to use replica
SELECT @@hostname, @@read_only;

-- Partitioning for large tables
ALTER TABLE orders PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Archive old data
CREATE TABLE orders_archive LIKE orders;
INSERT INTO orders_archive 
SELECT * FROM orders 
WHERE created_at < DATE_SUB(CURRENT_DATE, INTERVAL 2 YEAR);
```

## 8. Phase 6: Concurrent User Scenarios

### 8.1 Handle Race Conditions

Test and fix concurrency issues:

```sql
-- Scenario: Two users trying to buy the last item
-- Solution: Use SELECT FOR UPDATE

DELIMITER //
CREATE PROCEDURE BuyProductSafe(
    IN p_customer_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_result VARCHAR(100)
)
BEGIN
    DECLARE v_available_stock INT;
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_result = 'Purchase failed due to system error';
    END;
    
    START TRANSACTION;
    
    -- Lock the product row to prevent race conditions
    SELECT stock_quantity INTO v_available_stock
    FROM products 
    WHERE product_id = p_product_id 
    FOR UPDATE;
    
    IF v_available_stock >= p_quantity THEN
        -- Update inventory
        UPDATE products 
        SET stock_quantity = stock_quantity - p_quantity
        WHERE product_id = p_product_id;
        
        -- Create order (simplified)
        INSERT INTO orders (customer_id, total_amount, status)
        VALUES (p_customer_id, 0, 'pending');
        
        SET p_result = 'Purchase successful';
        COMMIT;
    ELSE
        SET p_result = 'Insufficient stock';
        ROLLBACK;
    END IF;
END //
DELIMITER ;
```

### 8.2 Load Testing

Create scripts to simulate concurrent users:

```sql
-- Create test scenarios
-- Simulate 100 concurrent users placing orders
-- Monitor for deadlocks and performance degradation

-- Check for deadlocks
SHOW ENGINE INNODB STATUS;

-- Monitor active connections
SHOW PROCESSLIST;

-- Check lock waits
SELECT * FROM information_schema.innodb_lock_waits;
```

## 9. Deliverables and Assessment

### 9.1 Required Deliverables

Submit the following:

1. **Complete Schema DDL** (CREATE statements for all tables)
2. **Sample Data Scripts** (INSERT statements or data generation procedures)
3. **Business Logic Procedures** (Stored procedures for core operations)
4. **Reporting Queries** (At least 10 business intelligence queries)
5. **Performance Analysis** (EXPLAIN plans and optimization strategies)
6. **Concurrency Testing** (Scripts and results demonstrating safe concurrent operations)
7. **Documentation** (ERD, assumptions, design decisions)

### 9.2 Evaluation Criteria

**Schema Design (25%)**
- Proper normalization and relationship modeling
- Appropriate data types and constraints
- Index design for performance
- Scalability considerations

**Implementation Quality (25%)**
- Code organization and readability
- Error handling and transaction management
- Business rule enforcement
- Data integrity maintenance

**Performance (25%)**
- Query optimization techniques
- Index usage and effectiveness
- Handling of large datasets
- Concurrent user support

**Business Requirements (25%)**
- Complete feature implementation
- Realistic business logic
- Comprehensive reporting capabilities
- User experience considerations

## 10. Extension Challenges

For advanced students, implement these additional features:

### 10.1 Advanced Features
- Multi-tenant architecture (support multiple stores)
- Real-time inventory synchronization
- Advanced pricing rules (bulk discounts, promotions)
- Recommendation engine based on purchase history
- Fraud detection patterns

### 10.2 Integration Scenarios
- REST API endpoints for mobile app
- Integration with external payment processors
- Data warehouse ETL processes
- Real-time analytics dashboard
- Microservices architecture considerations

## 11. Common Pitfalls to Avoid

1. **Over-normalization**: Don't normalize to the point where queries become too complex
2. **Under-indexing**: Missing indexes on foreign keys and search columns
3. **Ignoring Concurrency**: Not handling race conditions in inventory management
4. **Poor Error Handling**: Not implementing proper rollback mechanisms
5. **Unrealistic Data**: Using toy datasets that don't reveal performance issues
6. **Missing Business Logic**: Implementing technical features without business context

---

**Navigation**

[← Previous: Integration and Migration](10-03-integration-migration.md) | [Next → Project 2: Data Warehouse Implementation](project-02-data-warehouse.md)

_Last updated: 2025-01-21_ 