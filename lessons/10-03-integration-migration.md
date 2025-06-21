# Integration and Migration

> **Module 10 • Lesson 3**  
> Estimated time: 45 min | Difficulty: ★★★★★

## 1. Why this matters

Database integration and migration are critical skills for modern data engineering. Whether you're connecting multiple systems, migrating from legacy databases, implementing ETL pipelines, or building real-time data synchronization, these operations determine the success of data-driven applications. Poor integration leads to data inconsistencies, while failed migrations can cause extended downtime and data loss. Understanding how to design robust integration patterns, execute zero-downtime migrations, and handle data transformation at scale is essential for maintaining reliable, scalable data systems.

> **Need a refresher?** This lesson builds on concepts from [Data Modeling for Different Workloads](10-01-data-modeling-workloads.md) and [Development Phase Issues](09-01-development-phase-issues.md).

## 2. Key Concepts

- **ETL vs ELT**: Extract-Transform-Load vs Extract-Load-Transform patterns
- **Change Data Capture (CDC)**: Real-time data synchronization
- **Database migration strategies**: Big bang, blue-green, rolling migrations
- **Data transformation**: Schema mapping, data cleansing, format conversion
- **Integration patterns**: Batch processing, streaming, API-based integration
- **Cross-database compatibility**: Handling vendor-specific features

```mermaid
graph TD
    A["Source Systems"] --> B["Data Integration Layer"]
    B --> C["Transformation Engine"]
    C --> D["Target Systems"]
    
    E["Legacy Database"] --> F["Migration Strategy"]
    F --> G["New Database"]
    
    H["Real-time Sources"] --> I["Change Data Capture"]
    I --> J["Stream Processing"]
    J --> K["Multiple Targets"]
    
    B -.-> I
    C -.-> J
```

## 3. Deep Dive

### 3.1 ETL vs ELT Patterns

**Traditional ETL (Extract-Transform-Load):**
```sql
-- ETL Example: Data warehouse loading
-- 1. Extract from source systems
CREATE TABLE staging_orders AS
SELECT 
    order_id,
    customer_id,
    order_date,
    total_amount,
    'source_system_1' as source
FROM source1.orders
WHERE order_date >= CURDATE() - INTERVAL 1 DAY

UNION ALL

SELECT 
    order_ref as order_id,
    cust_id as customer_id,
    created_date as order_date,
    amount as total_amount,
    'source_system_2' as source
FROM source2.transactions
WHERE created_date >= CURDATE() - INTERVAL 1 DAY;

-- 2. Transform data
CREATE TABLE transformed_orders AS
SELECT 
    order_id,
    customer_id,
    order_date,
    total_amount,
    source,
    -- Data cleansing and standardization
    CASE 
        WHEN total_amount < 0 THEN 0
        ELSE total_amount
    END as clean_total,
    -- Derived fields
    YEAR(order_date) as order_year,
    MONTH(order_date) as order_month,
    CASE 
        WHEN total_amount >= 1000 THEN 'high_value'
        WHEN total_amount >= 100 THEN 'medium_value'
        ELSE 'low_value'
    END as order_category
FROM staging_orders
WHERE customer_id IS NOT NULL
  AND order_date IS NOT NULL
  AND total_amount > 0;

-- 3. Load into data warehouse
INSERT INTO warehouse.fact_orders (
    order_key, customer_key, date_key, amount, category, source_system
)
SELECT 
    t.order_id,
    c.customer_key,
    d.date_key,
    t.clean_total,
    t.order_category,
    t.source
FROM transformed_orders t
JOIN warehouse.dim_customers c ON t.customer_id = c.customer_id
JOIN warehouse.dim_dates d ON t.order_date = d.full_date;
```

**Modern ELT (Extract-Load-Transform):**
```sql
-- ELT Example: Load raw data first, transform in the target system
-- 1. Extract and Load raw data
CREATE TABLE raw_data.orders_source1 (
    order_id VARCHAR(50),
    customer_id VARCHAR(50),
    order_date DATETIME,
    total_amount DECIMAL(10,2),
    raw_json JSON,
    ingestion_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Load data as-is (minimal processing)
LOAD DATA INFILE '/data/orders_export.csv'
INTO TABLE raw_data.orders_source1
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- 2. Transform using SQL in the target system
CREATE VIEW analytics.clean_orders AS
SELECT 
    CAST(order_id AS UNSIGNED) as order_id,
    CAST(customer_id AS UNSIGNED) as customer_id,
    STR_TO_DATE(order_date, '%Y-%m-%d %H:%i:%s') as order_date,
    GREATEST(total_amount, 0) as total_amount,
    JSON_EXTRACT(raw_json, '$.payment_method') as payment_method,
    JSON_EXTRACT(raw_json, '$.shipping_address') as shipping_address
FROM raw_data.orders_source1
WHERE order_id REGEXP '^[0-9]+$'
  AND customer_id REGEXP '^[0-9]+$'
  AND total_amount IS NOT NULL;
```

### 3.2 Change Data Capture (CDC) Implementation

**Binary Log-based CDC (MySQL):**
```sql
-- Enable binary logging for CDC
-- In my.cnf:
-- log-bin=mysql-bin
-- binlog-format=ROW
-- binlog-row-image=FULL

-- Create CDC tracking table
CREATE TABLE cdc_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    operation ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    primary_key_value VARCHAR(255) NOT NULL,
    before_data JSON,
    after_data JSON,
    binlog_file VARCHAR(255),
    binlog_position BIGINT,
    timestamp TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
    processed BOOLEAN DEFAULT FALSE,
    INDEX idx_table_processed (table_name, processed),
    INDEX idx_timestamp (timestamp)
);

-- Trigger-based CDC for specific tables
DELIMITER //
CREATE TRIGGER products_cdc_insert
AFTER INSERT ON products
FOR EACH ROW
BEGIN
    INSERT INTO cdc_log (table_name, operation, primary_key_value, after_data)
    VALUES (
        'products',
        'INSERT',
        NEW.id,
        JSON_OBJECT(
            'id', NEW.id,
            'name', NEW.name,
            'price', NEW.price,
            'category', NEW.category,
            'updated_at', NEW.updated_at
        )
    );
END //

CREATE TRIGGER products_cdc_update
AFTER UPDATE ON products
FOR EACH ROW
BEGIN
    INSERT INTO cdc_log (table_name, operation, primary_key_value, before_data, after_data)
    VALUES (
        'products',
        'UPDATE',
        NEW.id,
        JSON_OBJECT(
            'id', OLD.id,
            'name', OLD.name,
            'price', OLD.price,
            'category', OLD.category,
            'updated_at', OLD.updated_at
        ),
        JSON_OBJECT(
            'id', NEW.id,
            'name', NEW.name,
            'price', NEW.price,
            'category', NEW.category,
            'updated_at', NEW.updated_at
        )
    );
END //

CREATE TRIGGER products_cdc_delete
AFTER DELETE ON products
FOR EACH ROW
BEGIN
    INSERT INTO cdc_log (table_name, operation, primary_key_value, before_data)
    VALUES (
        'products',
        'DELETE',
        OLD.id,
        JSON_OBJECT(
            'id', OLD.id,
            'name', OLD.name,
            'price', OLD.price,
            'category', OLD.category,
            'updated_at', OLD.updated_at
        )
    );
END //
DELIMITER ;

-- CDC processing procedure
DELIMITER //
CREATE PROCEDURE ProcessCDCBatch(IN batch_size INT)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_id BIGINT;
    DECLARE v_table_name VARCHAR(100);
    DECLARE v_operation VARCHAR(10);
    DECLARE v_primary_key VARCHAR(255);
    DECLARE v_after_data JSON;
    
    DECLARE cdc_cursor CURSOR FOR
        SELECT id, table_name, operation, primary_key_value, after_data
        FROM cdc_log
        WHERE processed = FALSE
        ORDER BY id
        LIMIT batch_size;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN cdc_cursor;
    
    read_loop: LOOP
        FETCH cdc_cursor INTO v_id, v_table_name, v_operation, v_primary_key, v_after_data;
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        -- Process the change (example: replicate to another database)
        CASE v_table_name
            WHEN 'products' THEN
                CALL ReplicateProductChange(v_operation, v_primary_key, v_after_data);
            WHEN 'orders' THEN
                CALL ReplicateOrderChange(v_operation, v_primary_key, v_after_data);
        END CASE;
        
        -- Mark as processed
        UPDATE cdc_log SET processed = TRUE WHERE id = v_id;
    END LOOP;
    
    CLOSE cdc_cursor;
END //
DELIMITER ;
```

### 3.3 Database Migration Strategies

**Blue-Green Migration:**
```sql
-- Blue-Green migration setup
-- Blue (current production)
CREATE DATABASE ecommerce_blue;

-- Green (new version)
CREATE DATABASE ecommerce_green;

-- Migration script
DELIMITER //
CREATE PROCEDURE BlueGreenMigration()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        -- Switch back to blue if migration fails
        -- Update application config to point to blue
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- 1. Copy current data to green database
    CREATE TABLE ecommerce_green.customers AS 
    SELECT * FROM ecommerce_blue.customers;
    
    CREATE TABLE ecommerce_green.orders AS
    SELECT * FROM ecommerce_blue.orders;
    
    -- 2. Apply schema changes in green
    ALTER TABLE ecommerce_green.customers 
    ADD COLUMN customer_tier ENUM('bronze', 'silver', 'gold') DEFAULT 'bronze';
    
    -- 3. Migrate data with transformations
    UPDATE ecommerce_green.customers 
    SET customer_tier = CASE 
        WHEN (SELECT COUNT(*) FROM ecommerce_green.orders o WHERE o.customer_id = customers.id) >= 10 THEN 'gold'
        WHEN (SELECT COUNT(*) FROM ecommerce_green.orders o WHERE o.customer_id = customers.id) >= 5 THEN 'silver'
        ELSE 'bronze'
    END;
    
    -- 4. Validate data integrity
    SET @blue_count = (SELECT COUNT(*) FROM ecommerce_blue.customers);
    SET @green_count = (SELECT COUNT(*) FROM ecommerce_green.customers);
    
    IF @blue_count != @green_count THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Data count mismatch';
    END IF;
    
    COMMIT;
    
    -- 5. Switch application to green (done externally)
    -- 6. Keep blue as backup for rollback
END //
DELIMITER ;
```

**Rolling Migration with Dual-Write:**
```sql
-- Rolling migration with dual-write pattern
CREATE TABLE migration_status (
    table_name VARCHAR(100) PRIMARY KEY,
    migration_phase ENUM('not_started', 'dual_write', 'read_new', 'complete') DEFAULT 'not_started',
    start_time TIMESTAMP NULL,
    completion_time TIMESTAMP NULL,
    last_migrated_id BIGINT DEFAULT 0
);

-- Dual-write trigger during migration
DELIMITER //
CREATE TRIGGER orders_dual_write
AFTER INSERT ON orders_old
FOR EACH ROW
BEGIN
    DECLARE migration_phase VARCHAR(20);
    
    SELECT mp.migration_phase INTO migration_phase
    FROM migration_status mp
    WHERE mp.table_name = 'orders';
    
    -- If in dual-write phase, also write to new table
    IF migration_phase IN ('dual_write', 'read_new') THEN
        INSERT INTO orders_new (
            id, customer_id, order_date, total, status, 
            -- New columns with default values
            order_source, customer_segment
        ) VALUES (
            NEW.id, NEW.customer_id, NEW.order_date, NEW.total, NEW.status,
            'legacy_migration',
            CASE 
                WHEN NEW.total >= 1000 THEN 'premium'
                WHEN NEW.total >= 100 THEN 'standard'
                ELSE 'basic'
            END
        );
    END IF;
END //
DELIMITER ;

-- Background migration process
DELIMITER //
CREATE PROCEDURE MigrateOrdersBatch(IN batch_size INT)
BEGIN
    DECLARE last_id BIGINT DEFAULT 0;
    DECLARE rows_migrated INT DEFAULT 0;
    
    -- Get last migrated ID
    SELECT last_migrated_id INTO last_id
    FROM migration_status
    WHERE table_name = 'orders';
    
    -- Migrate batch
    INSERT INTO orders_new (
        id, customer_id, order_date, total, status, order_source, customer_segment
    )
    SELECT 
        o.id, o.customer_id, o.order_date, o.total, o.status,
        'legacy_migration',
        CASE 
            WHEN o.total >= 1000 THEN 'premium'
            WHEN o.total >= 100 THEN 'standard'
            ELSE 'basic'
        END
    FROM orders_old o
    WHERE o.id > last_id
      AND o.id NOT IN (SELECT id FROM orders_new WHERE id = o.id)  -- Avoid duplicates
    ORDER BY o.id
    LIMIT batch_size;
    
    SET rows_migrated = ROW_COUNT();
    
    -- Update migration status
    IF rows_migrated > 0 THEN
        UPDATE migration_status
        SET last_migrated_id = (
            SELECT MAX(id) FROM orders_old WHERE id > last_id LIMIT batch_size
        )
        WHERE table_name = 'orders';
    END IF;
    
    SELECT rows_migrated as migrated_count;
END //
DELIMITER ;
```

### 3.4 Cross-Database Integration

**Database Federation Example:**
```sql
-- MySQL connecting to external databases
-- Enable FEDERATED storage engine
-- In my.cnf: federated

-- Create connection to PostgreSQL via ODBC
CREATE TABLE federated_postgres_customers (
    id INT NOT NULL,
    name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP
) ENGINE=FEDERATED
CONNECTION='mysql://user:password@postgres_host:5432/database/customers';

-- Create connection to SQL Server
CREATE TABLE federated_sqlserver_orders (
    order_id INT NOT NULL,
    customer_id INT,
    order_date DATE,
    total DECIMAL(10,2)
) ENGINE=FEDERATED
CONNECTION='mysql://user:password@sqlserver_host:1433/database/orders';

-- Cross-database query
SELECT 
    c.name as customer_name,
    c.email,
    COUNT(o.order_id) as total_orders,
    SUM(o.total) as total_spent
FROM federated_postgres_customers c
LEFT JOIN federated_sqlserver_orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email
HAVING total_orders > 0;

-- Data synchronization procedure
DELIMITER //
CREATE PROCEDURE SyncCustomerData()
BEGIN
    -- Sync from PostgreSQL to local MySQL
    INSERT INTO local_customers (id, name, email, created_at, sync_timestamp)
    SELECT 
        id, name, email, created_at, NOW()
    FROM federated_postgres_customers
    WHERE id NOT IN (SELECT id FROM local_customers)
    ON DUPLICATE KEY UPDATE
        name = VALUES(name),
        email = VALUES(email),
        sync_timestamp = NOW();
    
    -- Sync orders from SQL Server
    INSERT INTO local_orders (order_id, customer_id, order_date, total, sync_timestamp)
    SELECT 
        order_id, customer_id, order_date, total, NOW()
    FROM federated_sqlserver_orders
    WHERE order_id NOT IN (SELECT order_id FROM local_orders)
    ON DUPLICATE KEY UPDATE
        total = VALUES(total),
        sync_timestamp = NOW();
END //
DELIMITER ;
```

### 3.5 Data Transformation and Cleansing

**Advanced Data Transformation:**
```sql
-- Complex data transformation pipeline
CREATE TABLE raw_customer_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    raw_data JSON,
    source_system VARCHAR(50),
    ingestion_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed BOOLEAN DEFAULT FALSE
);

-- Data cleansing and standardization
DELIMITER //
CREATE PROCEDURE TransformCustomerData()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_id INT;
    DECLARE v_raw_data JSON;
    DECLARE v_source VARCHAR(50);
    
    DECLARE data_cursor CURSOR FOR
        SELECT id, raw_data, source_system
        FROM raw_customer_data
        WHERE processed = FALSE
        LIMIT 1000;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN data_cursor;
    
    transform_loop: LOOP
        FETCH data_cursor INTO v_id, v_raw_data, v_source;
        IF done THEN
            LEAVE transform_loop;
        END IF;
        
        -- Transform and insert clean data
        INSERT INTO clean_customers (
            external_id,
            name,
            email,
            phone,
            address,
            city,
            state,
            postal_code,
            country,
            source_system,
            data_quality_score
        )
        SELECT 
            JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.id')) as external_id,
            -- Name standardization
            TRIM(UPPER(COALESCE(
                JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.full_name')),
                CONCAT(
                    JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.first_name')), ' ',
                    JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.last_name'))
                )
            ))) as name,
            -- Email validation and standardization
            CASE 
                WHEN JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.email')) REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
                THEN LOWER(JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.email')))
                ELSE NULL
            END as email,
            -- Phone number standardization
            REGEXP_REPLACE(
                JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.phone')),
                '[^0-9]', ''
            ) as phone,
            JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.address')) as address,
            JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.city')) as city,
            JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.state')) as state,
            -- Postal code standardization
            CASE 
                WHEN v_source = 'US_SYSTEM' THEN 
                    REGEXP_REPLACE(JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.zip')), '[^0-9-]', '')
                ELSE JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.postal_code'))
            END as postal_code,
            COALESCE(
                JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.country')),
                'US'
            ) as country,
            v_source as source_system,
            -- Data quality scoring
            (
                (CASE WHEN JSON_UNQUOTE(JSON_EXTRACT(v_raw_data, '$.email')) REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN 25 ELSE 0 END) +
                (CASE WHEN JSON_EXTRACT(v_raw_data, '$.phone') IS NOT NULL THEN 25 ELSE 0 END) +
                (CASE WHEN JSON_EXTRACT(v_raw_data, '$.address') IS NOT NULL THEN 25 ELSE 0 END) +
                (CASE WHEN JSON_EXTRACT(v_raw_data, '$.full_name') IS NOT NULL OR 
                           (JSON_EXTRACT(v_raw_data, '$.first_name') IS NOT NULL AND 
                            JSON_EXTRACT(v_raw_data, '$.last_name') IS NOT NULL) THEN 25 ELSE 0 END)
            ) as data_quality_score;
        
        -- Mark as processed
        UPDATE raw_customer_data SET processed = TRUE WHERE id = v_id;
    END LOOP;
    
    CLOSE data_cursor;
END //
DELIMITER ;
```

## 4. Hands-On Practice

Let's implement a complete integration and migration scenario:

```sql
-- Scenario: Migrating from legacy system to modern microservices architecture

-- 1. Legacy monolithic database
CREATE DATABASE legacy_ecommerce;
USE legacy_ecommerce;

CREATE TABLE legacy_customers (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(200),
    customer_email VARCHAR(255),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    registration_date DATETIME,
    customer_status CHAR(1), -- A=Active, I=Inactive, S=Suspended
    total_orders INT DEFAULT 0,
    total_spent DECIMAL(12,2) DEFAULT 0
);

CREATE TABLE legacy_orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    order_datetime DATETIME,
    order_amount DECIMAL(10,2),
    order_status VARCHAR(20),
    payment_method VARCHAR(50),
    shipping_address TEXT,
    FOREIGN KEY (customer_id) REFERENCES legacy_customers(customer_id)
);

-- 2. New microservices databases
CREATE DATABASE customer_service;
CREATE DATABASE order_service;
CREATE DATABASE analytics_service;

-- Customer microservice schema
USE customer_service;
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT (UUID()),
    external_id VARCHAR(50) UNIQUE,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    version INT DEFAULT 1,
    INDEX idx_external_id (external_id),
    INDEX idx_email (email)
);

CREATE TABLE customer_addresses (
    id UUID PRIMARY KEY DEFAULT (UUID()),
    customer_id UUID NOT NULL,
    address_type ENUM('billing', 'shipping') NOT NULL,
    street_address VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    postal_code VARCHAR(20),
    country VARCHAR(50) DEFAULT 'US',
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE
);

-- Order microservice schema  
USE order_service;
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT (UUID()),
    external_id VARCHAR(50) UNIQUE,
    customer_id UUID NOT NULL,
    status ENUM('pending', 'confirmed', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(12,2) NOT NULL,
    currency CHAR(3) DEFAULT 'USD',
    payment_method VARCHAR(50),
    shipping_address JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_customer_id (customer_id),
    INDEX idx_external_id (external_id),
    INDEX idx_status_created (status, created_at)
);

-- Analytics service schema
USE analytics_service;
CREATE TABLE customer_analytics (
    customer_id UUID PRIMARY KEY,
    total_orders INT DEFAULT 0,
    total_spent DECIMAL(12,2) DEFAULT 0,
    average_order_value DECIMAL(10,2) DEFAULT 0,
    first_order_date DATE,
    last_order_date DATE,
    customer_lifetime_days INT DEFAULT 0,
    customer_segment ENUM('new', 'active', 'at_risk', 'churned') DEFAULT 'new',
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 3. Migration procedures
USE legacy_ecommerce;

DELIMITER //
CREATE PROCEDURE MigrateCustomers(IN batch_size INT)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_customer_id INT;
    DECLARE v_name VARCHAR(200);
    DECLARE v_email VARCHAR(255);
    DECLARE v_phone VARCHAR(20);
    DECLARE v_address TEXT;
    DECLARE v_status CHAR(1);
    DECLARE v_reg_date DATETIME;
    
    DECLARE customer_cursor CURSOR FOR
        SELECT customer_id, customer_name, customer_email, customer_phone, 
               customer_address, customer_status, registration_date
        FROM legacy_customers
        WHERE customer_id NOT IN (
            SELECT CAST(external_id AS UNSIGNED)
            FROM customer_service.customers 
            WHERE external_id IS NOT NULL
        )
        LIMIT batch_size;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN customer_cursor;
    
    migrate_loop: LOOP
        FETCH customer_cursor INTO v_customer_id, v_name, v_email, v_phone, 
                                   v_address, v_status, v_reg_date;
        IF done THEN
            LEAVE migrate_loop;
        END IF;
        
        -- Parse name into first and last
        SET @first_name = SUBSTRING_INDEX(v_name, ' ', 1);
        SET @last_name = CASE 
            WHEN LOCATE(' ', v_name) > 0 THEN SUBSTRING(v_name, LOCATE(' ', v_name) + 1)
            ELSE ''
        END;
        
        -- Convert status
        SET @new_status = CASE v_status
            WHEN 'A' THEN 'active'
            WHEN 'I' THEN 'inactive'
            WHEN 'S' THEN 'suspended'
            ELSE 'inactive'
        END;
        
        -- Insert into new customer service
        INSERT INTO customer_service.customers (
            external_id, email, first_name, last_name, phone, status, created_at
        ) VALUES (
            v_customer_id, v_email, @first_name, @last_name, v_phone, @new_status, v_reg_date
        );
        
        -- Insert address if available
        IF v_address IS NOT NULL AND LENGTH(v_address) > 0 THEN
            INSERT INTO customer_service.customer_addresses (
                customer_id, address_type, street_address, is_default
            ) VALUES (
                (SELECT id FROM customer_service.customers WHERE external_id = v_customer_id),
                'billing',
                v_address,
                TRUE
            );
        END IF;
    END LOOP;
    
    CLOSE customer_cursor;
END //

CREATE PROCEDURE MigrateOrders(IN batch_size INT)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_order_id INT;
    DECLARE v_customer_id INT;
    DECLARE v_order_date DATETIME;
    DECLARE v_amount DECIMAL(10,2);
    DECLARE v_status VARCHAR(20);
    DECLARE v_payment VARCHAR(50);
    DECLARE v_shipping TEXT;
    
    DECLARE order_cursor CURSOR FOR
        SELECT order_id, customer_id, order_datetime, order_amount, 
               order_status, payment_method, shipping_address
        FROM legacy_orders
        WHERE order_id NOT IN (
            SELECT CAST(external_id AS UNSIGNED)
            FROM order_service.orders 
            WHERE external_id IS NOT NULL
        )
        LIMIT batch_size;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN order_cursor;
    
    migrate_loop: LOOP
        FETCH order_cursor INTO v_order_id, v_customer_id, v_order_date, 
                                v_amount, v_status, v_payment, v_shipping;
        IF done THEN
            LEAVE migrate_loop;
        END IF;
        
        -- Convert status
        SET @new_status = CASE LOWER(v_status)
            WHEN 'new' THEN 'pending'
            WHEN 'processing' THEN 'confirmed'
            WHEN 'shipped' THEN 'shipped'
            WHEN 'completed' THEN 'delivered'
            WHEN 'cancelled' THEN 'cancelled'
            ELSE 'pending'
        END;
        
        -- Get new customer UUID
        SET @customer_uuid = (
            SELECT id FROM customer_service.customers 
            WHERE external_id = v_customer_id
        );
        
        IF @customer_uuid IS NOT NULL THEN
            INSERT INTO order_service.orders (
                external_id, customer_id, status, total_amount, 
                payment_method, shipping_address, created_at
            ) VALUES (
                v_order_id, @customer_uuid, @new_status, v_amount,
                v_payment, JSON_OBJECT('address', v_shipping), v_order_date
            );
        END IF;
    END LOOP;
    
    CLOSE order_cursor;
END //
DELIMITER ;

-- 4. Analytics calculation
USE analytics_service;
DELIMITER //
CREATE PROCEDURE UpdateCustomerAnalytics()
BEGIN
    INSERT INTO customer_analytics (
        customer_id, total_orders, total_spent, average_order_value,
        first_order_date, last_order_date, customer_lifetime_days, customer_segment
    )
    SELECT 
        c.id as customer_id,
        COALESCE(order_stats.total_orders, 0) as total_orders,
        COALESCE(order_stats.total_spent, 0) as total_spent,
        COALESCE(order_stats.avg_order_value, 0) as average_order_value,
        order_stats.first_order_date,
        order_stats.last_order_date,
        COALESCE(DATEDIFF(CURDATE(), order_stats.first_order_date), 0) as customer_lifetime_days,
        CASE 
            WHEN order_stats.last_order_date IS NULL THEN 'new'
            WHEN order_stats.last_order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY) THEN 'active'
            WHEN order_stats.last_order_date >= DATE_SUB(CURDATE(), INTERVAL 90 DAY) THEN 'at_risk'
            ELSE 'churned'
        END as customer_segment
    FROM customer_service.customers c
    LEFT JOIN (
        SELECT 
            customer_id,
            COUNT(*) as total_orders,
            SUM(total_amount) as total_spent,
            AVG(total_amount) as avg_order_value,
            MIN(DATE(created_at)) as first_order_date,
            MAX(DATE(created_at)) as last_order_date
        FROM order_service.orders
        WHERE status IN ('delivered', 'shipped')
        GROUP BY customer_id
    ) order_stats ON c.id = order_stats.customer_id
    ON DUPLICATE KEY UPDATE
        total_orders = VALUES(total_orders),
        total_spent = VALUES(total_spent),
        average_order_value = VALUES(average_order_value),
        first_order_date = VALUES(first_order_date),
        last_order_date = VALUES(last_order_date),
        customer_lifetime_days = VALUES(customer_lifetime_days),
        customer_segment = VALUES(customer_segment);
END //
DELIMITER ;
```

**Practice Exercises:**

1. **Implement real-time CDC** using MySQL binary logs
2. **Design a zero-downtime migration** for a high-traffic application  
3. **Create an ETL pipeline** that handles schema evolution
4. **Build cross-database federation** for reporting across multiple systems

## 5. Common Pitfalls

### 5.1 Inadequate Migration Testing
**Problem**: Not testing migration procedures thoroughly
**Solution**: Test with production-like data volumes and implement rollback procedures

### 5.2 Data Loss During Migration
**Problem**: Not handling edge cases and data validation
**Solution**: Implement comprehensive data validation and maintain audit trails

### 5.3 Performance Impact During Migration
**Problem**: Migration causing production slowdowns
**Solution**: Use batch processing, off-peak scheduling, and resource throttling

### 5.4 Schema Drift
**Problem**: Source and target schemas getting out of sync
**Solution**: Version control schemas and implement automated drift detection

### 5.5 Ignoring Data Dependencies
**Problem**: Breaking referential integrity during migration
**Solution**: Map all dependencies and migrate in correct order

## 6. Knowledge Check

<details>
<summary>1. What's the difference between ETL and ELT approaches?</summary>

ETL transforms data before loading into the target system, requiring separate transformation infrastructure. ELT loads raw data first, then transforms it in the target system, leveraging the target's processing power and allowing for more flexible transformations.
</details>

<details>
<summary>2. When would you use Change Data Capture (CDC)?</summary>

Use CDC for real-time data synchronization, event-driven architectures, maintaining data lakes, replication between systems, and audit trails. It's essential when you need to track and propagate data changes with minimal latency.
</details>

<details>
<summary>3. What are the advantages of blue-green deployment for database migrations?</summary>

Blue-green deployment allows for instant rollback, reduced downtime, thorough testing of the new environment, and the ability to validate the migration before switching traffic. It requires more resources but provides maximum safety.
</details>

<details>
<summary>4. How do you handle schema evolution in a microservices architecture?</summary>

Use versioned APIs, backward-compatible schema changes, event sourcing for audit trails, database per service pattern, and implement proper service contracts. Plan for gradual migration and maintain compatibility during transitions.
</details>

<details>
<summary>5. What are the key considerations for cross-database integration?</summary>

Consider data consistency requirements, network latency, security and access control, data format compatibility, transaction boundaries, error handling, and monitoring. Choose appropriate integration patterns based on consistency and performance requirements.
</details>

## 7. Further Reading

- [Database Migration Strategies - Martin Fowler](https://martinfowler.com/articles/evodb.html)
- [Change Data Capture Patterns](https://www.confluent.io/blog/change-data-capture-pattern/)
- [ETL vs ELT: What's the Difference?](https://www.integrate.io/blog/etl-vs-elt/)
- [Microservices Data Management](https://microservices.io/patterns/data/)
- [MySQL Binary Log and Replication](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)
- [Zero Downtime Database Migrations](https://blog.rainforestqa.com/2014-06-27-zero-downtime-database-migrations/)

---

**Navigation**

[← Previous: Security and Compliance](10-02-security-compliance.md) | [Next → E-commerce Database Design Project](project-01-ecommerce-database.md)

_Last updated: 2025-01-21_ 