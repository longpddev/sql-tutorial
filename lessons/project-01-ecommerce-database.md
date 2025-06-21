# Project 1: E-commerce Database Design

> **Assessment Project • Comprehensive Implementation**  
> Estimated time: 8-12 hours | Difficulty: ★★★★☆

## 1. Project Overview

Design and implement a complete e-commerce database system that handles products, customers, orders, inventory, and analytics. This project integrates concepts from all previous modules: schema design, performance optimization, security, transactions, and real-world operational concerns. You'll build a system that could handle a medium-scale online retail business with thousands of products and customers.

**Learning Objectives:**
- Apply normalization principles while balancing performance needs
- Implement complex business logic with stored procedures and triggers
- Design for scalability and performance
- Handle real-world constraints like inventory management and pricing
- Create comprehensive reporting and analytics capabilities

> **Prerequisites:** Complete Modules 1-10, especially [Schema Design Patterns](05-01-schema-design-patterns.md), [Performance Patterns](05-03-performance-patterns.md), and [ACID Properties](06-01-acid-properties.md).

## 2. Business Requirements

### 2.1 Core Functionality
- **Product Management**: Categories, variants, pricing, inventory tracking
- **Customer Management**: Registration, profiles, addresses, preferences  
- **Order Processing**: Shopping cart, checkout, payment, fulfillment
- **Inventory Control**: Stock levels, reservations, restocking
- **Pricing**: Base prices, discounts, promotions, tax calculation
- **Analytics**: Sales reporting, customer insights, inventory analytics

### 2.2 Business Rules
- Products can have multiple variants (size, color, etc.)
- Customers can have multiple shipping addresses
- Orders must reserve inventory until payment confirmation
- Promotional discounts have time windows and usage limits
- Abandoned carts should be tracked for recovery campaigns
- All financial transactions must be auditable

### 2.3 Performance Requirements
- Support 10,000+ concurrent users
- Order processing < 2 seconds
- Product search < 500ms
- Analytics queries < 5 seconds
- 99.9% uptime requirement

## 3. Database Design

### 3.1 Core Schema Implementation

```sql
-- Create the e-commerce database
CREATE DATABASE ecommerce_system;
USE ecommerce_system;

-- Enable strict mode for data integrity
SET sql_mode = 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';

-- Categories with hierarchical structure
CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_id INT NULL,
    description TEXT,
    image_url VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_id) REFERENCES categories(id),
    INDEX idx_parent_active (parent_id, is_active),
    INDEX idx_slug (slug)
);

-- Products with comprehensive attributes
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    short_description TEXT,
    long_description LONGTEXT,
    category_id INT NOT NULL,
    brand VARCHAR(100),
    weight DECIMAL(8,3), -- in kg
    dimensions JSON, -- {"length": 10, "width": 5, "height": 3}
    base_price DECIMAL(10,2) NOT NULL,
    cost_price DECIMAL(10,2),
    tax_rate DECIMAL(5,4) DEFAULT 0.0000,
    is_active BOOLEAN DEFAULT TRUE,
    is_featured BOOLEAN DEFAULT FALSE,
    meta_title VARCHAR(200),
    meta_description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id),
    INDEX idx_category_active (category_id, is_active),
    INDEX idx_sku (sku),
    INDEX idx_slug (slug),
    INDEX idx_featured (is_featured, is_active),
    FULLTEXT idx_search (name, short_description, brand)
);

-- Product variants (size, color, etc.)
CREATE TABLE product_variants (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL, -- "Large Red"
    attributes JSON NOT NULL, -- {"size": "L", "color": "red"}
    price_adjustment DECIMAL(10,2) DEFAULT 0.00,
    weight_adjustment DECIMAL(8,3) DEFAULT 0.000,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    INDEX idx_product_active (product_id, is_active),
    INDEX idx_sku (sku)
);

-- Inventory management
CREATE TABLE inventory (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    variant_id INT NULL,
    warehouse_location VARCHAR(100) DEFAULT 'MAIN',
    quantity_available INT NOT NULL DEFAULT 0,
    quantity_reserved INT NOT NULL DEFAULT 0,
    reorder_level INT DEFAULT 10,
    reorder_quantity INT DEFAULT 50,
    last_restocked_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (variant_id) REFERENCES product_variants(id),
    UNIQUE KEY unique_inventory (product_id, variant_id, warehouse_location),
    INDEX idx_low_stock (reorder_level, quantity_available)
);

-- Customer management
CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    gender ENUM('M', 'F', 'O', 'N') NULL, -- Male, Female, Other, Not specified
    email_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    customer_group ENUM('regular', 'vip', 'wholesale') DEFAULT 'regular',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP NULL,
    INDEX idx_email (email),
    INDEX idx_group_active (customer_group, is_active)
);

-- Customer addresses
CREATE TABLE customer_addresses (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    type ENUM('billing', 'shipping') NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    company VARCHAR(100),
    address_line_1 VARCHAR(255) NOT NULL,
    address_line_2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country CHAR(2) NOT NULL DEFAULT 'US',
    phone VARCHAR(20),
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE,
    INDEX idx_customer_type (customer_id, type),
    INDEX idx_customer_default (customer_id, is_default)
);

-- Shopping carts
CREATE TABLE shopping_carts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NULL, -- NULL for guest carts
    session_id VARCHAR(128) NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    expires_at TIMESTAMP DEFAULT (CURRENT_TIMESTAMP + INTERVAL 30 DAY),
    INDEX idx_customer (customer_id),
    INDEX idx_session (session_id),
    INDEX idx_expires (expires_at)
);

-- Cart items
CREATE TABLE cart_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    cart_id INT NOT NULL,
    product_id INT NOT NULL,
    variant_id INT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price DECIMAL(10,2) NOT NULL,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (cart_id) REFERENCES shopping_carts(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (variant_id) REFERENCES product_variants(id),
    UNIQUE KEY unique_cart_item (cart_id, product_id, variant_id),
    INDEX idx_cart (cart_id)
);

-- Orders
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_number VARCHAR(20) UNIQUE NOT NULL,
    customer_id INT NULL, -- NULL for guest orders
    status ENUM('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded') DEFAULT 'pending',
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    shipping_amount DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    discount_amount DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    total_amount DECIMAL(10,2) NOT NULL,
    currency CHAR(3) DEFAULT 'USD',
    payment_status ENUM('pending', 'paid', 'failed', 'refunded') DEFAULT 'pending',
    payment_method VARCHAR(50),
    shipping_method VARCHAR(50),
    billing_address JSON NOT NULL,
    shipping_address JSON NOT NULL,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    shipped_at TIMESTAMP NULL,
    delivered_at TIMESTAMP NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    INDEX idx_customer (customer_id),
    INDEX idx_status (status),
    INDEX idx_order_number (order_number),
    INDEX idx_created_at (created_at)
);

-- Order items
CREATE TABLE order_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    variant_id INT NULL,
    product_name VARCHAR(200) NOT NULL, -- Snapshot for historical accuracy
    product_sku VARCHAR(50) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (variant_id) REFERENCES product_variants(id),
    INDEX idx_order (order_id),
    INDEX idx_product (product_id)
);

-- Promotions and discounts
CREATE TABLE promotions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    type ENUM('percentage', 'fixed_amount', 'free_shipping') NOT NULL,
    value DECIMAL(10,2) NOT NULL,
    minimum_order_amount DECIMAL(10,2) DEFAULT 0.00,
    usage_limit INT NULL, -- NULL = unlimited
    usage_count INT DEFAULT 0,
    per_customer_limit INT DEFAULT 1,
    starts_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_code_active (code, is_active),
    INDEX idx_dates (starts_at, expires_at)
);

-- Track promotion usage
CREATE TABLE promotion_usage (
    id INT AUTO_INCREMENT PRIMARY KEY,
    promotion_id INT NOT NULL,
    order_id INT NOT NULL,
    customer_id INT NULL,
    discount_amount DECIMAL(10,2) NOT NULL,
    used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (promotion_id) REFERENCES promotions(id),
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    UNIQUE KEY unique_order_promotion (order_id, promotion_id),
    INDEX idx_promotion (promotion_id),
    INDEX idx_customer (customer_id)
);
```

### 3.2 Business Logic Implementation

```sql
-- Stored procedures for core business operations

DELIMITER //

-- Add item to cart with inventory check
CREATE PROCEDURE AddToCart(
    IN p_customer_id INT,
    IN p_session_id VARCHAR(128),
    IN p_product_id INT,
    IN p_variant_id INT,
    IN p_quantity INT,
    OUT p_success BOOLEAN,
    OUT p_message VARCHAR(255)
)
BEGIN
    DECLARE v_cart_id INT;
    DECLARE v_available_qty INT DEFAULT 0;
    DECLARE v_current_qty INT DEFAULT 0;
    DECLARE v_unit_price DECIMAL(10,2);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
        SET p_message = 'An error occurred while adding item to cart';
    END;
    
    START TRANSACTION;
    
    -- Check inventory availability
    SELECT COALESCE(quantity_available - quantity_reserved, 0)
    INTO v_available_qty
    FROM inventory
    WHERE product_id = p_product_id 
      AND (variant_id = p_variant_id OR (variant_id IS NULL AND p_variant_id IS NULL));
    
    -- Get current price
    SELECT 
        CASE 
            WHEN p_variant_id IS NOT NULL THEN p.base_price + pv.price_adjustment
            ELSE p.base_price
        END
    INTO v_unit_price
    FROM products p
    LEFT JOIN product_variants pv ON p.id = pv.product_id AND pv.id = p_variant_id
    WHERE p.id = p_product_id AND p.is_active = TRUE;
    
    IF v_unit_price IS NULL THEN
        SET p_success = FALSE;
        SET p_message = 'Product not found or inactive';
        ROLLBACK;
    ELSEIF v_available_qty < p_quantity THEN
        SET p_success = FALSE;
        SET p_message = CONCAT('Only ', v_available_qty, ' items available in stock');
        ROLLBACK;
    ELSE
        -- Get or create cart
        SELECT id INTO v_cart_id
        FROM shopping_carts
        WHERE (customer_id = p_customer_id OR session_id = p_session_id)
          AND expires_at > NOW()
        ORDER BY updated_at DESC
        LIMIT 1;
        
        IF v_cart_id IS NULL THEN
            INSERT INTO shopping_carts (customer_id, session_id)
            VALUES (p_customer_id, p_session_id);
            SET v_cart_id = LAST_INSERT_ID();
        END IF;
        
        -- Check if item already in cart
        SELECT quantity INTO v_current_qty
        FROM cart_items
        WHERE cart_id = v_cart_id 
          AND product_id = p_product_id 
          AND (variant_id = p_variant_id OR (variant_id IS NULL AND p_variant_id IS NULL));
        
        IF v_current_qty IS NULL THEN
            -- Add new item
            INSERT INTO cart_items (cart_id, product_id, variant_id, quantity, unit_price)
            VALUES (v_cart_id, p_product_id, p_variant_id, p_quantity, v_unit_price);
        ELSE
            -- Update existing item
            UPDATE cart_items
            SET quantity = v_current_qty + p_quantity,
                unit_price = v_unit_price
            WHERE cart_id = v_cart_id 
              AND product_id = p_product_id 
              AND (variant_id = p_variant_id OR (variant_id IS NULL AND p_variant_id IS NULL));
        END IF;
        
        -- Update cart timestamp
        UPDATE shopping_carts SET updated_at = NOW() WHERE id = v_cart_id;
        
        SET p_success = TRUE;
        SET p_message = 'Item added to cart successfully';
        COMMIT;
    END IF;
END //

-- Process order with inventory reservation
CREATE PROCEDURE ProcessOrder(
    IN p_customer_id INT,
    IN p_cart_id INT,
    IN p_billing_address JSON,
    IN p_shipping_address JSON,
    IN p_payment_method VARCHAR(50),
    IN p_shipping_method VARCHAR(50),
    IN p_promotion_code VARCHAR(50),
    OUT p_order_id INT,
    OUT p_success BOOLEAN,
    OUT p_message VARCHAR(255)
)
BEGIN
    DECLARE v_subtotal DECIMAL(10,2) DEFAULT 0.00;
    DECLARE v_tax_amount DECIMAL(10,2) DEFAULT 0.00;
    DECLARE v_shipping_amount DECIMAL(10,2) DEFAULT 0.00;
    DECLARE v_discount_amount DECIMAL(10,2) DEFAULT 0.00;
    DECLARE v_total_amount DECIMAL(10,2) DEFAULT 0.00;
    DECLARE v_order_number VARCHAR(20);
    DECLARE v_promotion_id INT;
    DECLARE done INT DEFAULT FALSE;
    
    DECLARE item_cursor CURSOR FOR
        SELECT ci.product_id, ci.variant_id, ci.quantity, ci.unit_price
        FROM cart_items ci
        WHERE ci.cart_id = p_cart_id;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
        SET p_message = 'Order processing failed';
    END;
    
    START TRANSACTION;
    
    -- Calculate subtotal
    SELECT SUM(quantity * unit_price) INTO v_subtotal
    FROM cart_items
    WHERE cart_id = p_cart_id;
    
    -- Apply promotion if provided
    IF p_promotion_code IS NOT NULL THEN
        SELECT id INTO v_promotion_id
        FROM promotions
        WHERE code = p_promotion_code
          AND is_active = TRUE
          AND starts_at <= NOW()
          AND expires_at > NOW()
          AND (usage_limit IS NULL OR usage_count < usage_limit)
          AND v_subtotal >= minimum_order_amount;
        
        IF v_promotion_id IS NOT NULL THEN
            SELECT 
                CASE type
                    WHEN 'percentage' THEN v_subtotal * (value / 100)
                    WHEN 'fixed_amount' THEN LEAST(value, v_subtotal)
                    WHEN 'free_shipping' THEN 0
                END
            INTO v_discount_amount
            FROM promotions
            WHERE id = v_promotion_id;
        END IF;
    END IF;
    
    -- Calculate shipping (simplified)
    SET v_shipping_amount = CASE p_shipping_method
        WHEN 'standard' THEN 5.99
        WHEN 'express' THEN 12.99
        WHEN 'overnight' THEN 24.99
        ELSE 0.00
    END;
    
    -- Apply free shipping promotion
    IF v_promotion_id IS NOT NULL THEN
        SELECT type INTO @promo_type FROM promotions WHERE id = v_promotion_id;
        IF @promo_type = 'free_shipping' THEN
            SET v_discount_amount = v_shipping_amount;
            SET v_shipping_amount = 0.00;
        END IF;
    END IF;
    
    -- Calculate tax (simplified - 8.5% rate)
    SET v_tax_amount = (v_subtotal - v_discount_amount) * 0.085;
    
    -- Calculate total
    SET v_total_amount = v_subtotal + v_tax_amount + v_shipping_amount - v_discount_amount;
    
    -- Generate order number
    SET v_order_number = CONCAT('ORD-', DATE_FORMAT(NOW(), '%Y%m%d'), '-', LPAD(CONNECTION_ID(), 6, '0'));
    
    -- Create order
    INSERT INTO orders (
        order_number, customer_id, subtotal, tax_amount, shipping_amount,
        discount_amount, total_amount, payment_method, shipping_method,
        billing_address, shipping_address
    ) VALUES (
        v_order_number, p_customer_id, v_subtotal, v_tax_amount, v_shipping_amount,
        v_discount_amount, v_total_amount, p_payment_method, p_shipping_method,
        p_billing_address, p_shipping_address
    );
    
    SET p_order_id = LAST_INSERT_ID();
    
    -- Copy cart items to order items and reserve inventory
    OPEN item_cursor;
    item_loop: LOOP
        FETCH item_cursor INTO @product_id, @variant_id, @quantity, @unit_price;
        IF done THEN
            LEAVE item_loop;
        END IF;
        
        -- Check inventory availability
        SELECT quantity_available - quantity_reserved INTO @available
        FROM inventory
        WHERE product_id = @product_id 
          AND (variant_id = @variant_id OR (variant_id IS NULL AND @variant_id IS NULL));
        
        IF @available < @quantity THEN
            SET p_success = FALSE;
            SET p_message = CONCAT('Insufficient inventory for product ID ', @product_id);
            ROLLBACK;
            LEAVE item_loop;
        END IF;
        
        -- Reserve inventory
        UPDATE inventory
        SET quantity_reserved = quantity_reserved + @quantity
        WHERE product_id = @product_id 
          AND (variant_id = @variant_id OR (variant_id IS NULL AND @variant_id IS NULL));
        
        -- Get product details for order item
        SELECT p.name, COALESCE(pv.sku, p.sku)
        INTO @product_name, @product_sku
        FROM products p
        LEFT JOIN product_variants pv ON p.id = pv.product_id AND pv.id = @variant_id
        WHERE p.id = @product_id;
        
        -- Create order item
        INSERT INTO order_items (
            order_id, product_id, variant_id, product_name, product_sku,
            quantity, unit_price, total_price
        ) VALUES (
            p_order_id, @product_id, @variant_id, @product_name, @product_sku,
            @quantity, @unit_price, @quantity * @unit_price
        );
    END LOOP;
    CLOSE item_cursor;
    
    -- Record promotion usage
    IF v_promotion_id IS NOT NULL AND p_success != FALSE THEN
        INSERT INTO promotion_usage (promotion_id, order_id, customer_id, discount_amount)
        VALUES (v_promotion_id, p_order_id, p_customer_id, v_discount_amount);
        
        UPDATE promotions
        SET usage_count = usage_count + 1
        WHERE id = v_promotion_id;
    END IF;
    
    -- Clear cart
    DELETE FROM cart_items WHERE cart_id = p_cart_id;
    
    IF p_success != FALSE THEN
        SET p_success = TRUE;
        SET p_message = 'Order processed successfully';
        COMMIT;
    END IF;
END //

DELIMITER ;
```

### 3.3 Analytics and Reporting

```sql
-- Create analytics views and procedures

-- Sales analytics view
CREATE VIEW sales_analytics AS
SELECT 
    DATE(o.created_at) as order_date,
    COUNT(o.id) as total_orders,
    SUM(o.total_amount) as total_revenue,
    AVG(o.total_amount) as average_order_value,
    SUM(oi.quantity) as total_items_sold,
    COUNT(DISTINCT o.customer_id) as unique_customers
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.status NOT IN ('cancelled', 'refunded')
GROUP BY DATE(o.created_at);

-- Product performance view
CREATE VIEW product_performance AS
SELECT 
    p.id,
    p.name,
    p.sku,
    c.name as category_name,
    COUNT(oi.id) as times_ordered,
    SUM(oi.quantity) as total_quantity_sold,
    SUM(oi.total_price) as total_revenue,
    AVG(oi.unit_price) as average_selling_price,
    i.quantity_available as current_stock
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.id AND o.status NOT IN ('cancelled', 'refunded')
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN inventory i ON p.id = i.product_id AND i.variant_id IS NULL
GROUP BY p.id, p.name, p.sku, c.name, i.quantity_available;

-- Customer analytics view
CREATE VIEW customer_analytics AS
SELECT 
    c.id,
    c.email,
    c.first_name,
    c.last_name,
    c.customer_group,
    COUNT(o.id) as total_orders,
    COALESCE(SUM(o.total_amount), 0) as lifetime_value,
    COALESCE(AVG(o.total_amount), 0) as average_order_value,
    MIN(o.created_at) as first_order_date,
    MAX(o.created_at) as last_order_date,
    DATEDIFF(NOW(), MAX(o.created_at)) as days_since_last_order
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY c.id, c.email, c.first_name, c.last_name, c.customer_group;

-- Inventory alerts view
CREATE VIEW inventory_alerts AS
SELECT 
    p.name as product_name,
    p.sku,
    COALESCE(pv.name, 'Default') as variant_name,
    i.quantity_available,
    i.quantity_reserved,
    i.reorder_level,
    i.warehouse_location,
    CASE 
        WHEN i.quantity_available <= i.reorder_level THEN 'LOW_STOCK'
        WHEN i.quantity_available = 0 THEN 'OUT_OF_STOCK'
        ELSE 'OK'
    END as alert_level
FROM inventory i
JOIN products p ON i.product_id = p.id
LEFT JOIN product_variants pv ON i.variant_id = pv.id
WHERE i.quantity_available <= i.reorder_level
ORDER BY i.quantity_available ASC;
```

## 4. Implementation Tasks

### Task 1: Schema Implementation (2 hours)
1. Create all tables with proper constraints and indexes
2. Insert sample data for categories, products, and customers
3. Verify referential integrity and constraint enforcement

### Task 2: Business Logic (3 hours)
1. Implement the AddToCart stored procedure
2. Implement the ProcessOrder stored procedure
3. Create additional procedures for inventory management
4. Test all procedures with various scenarios

### Task 3: Performance Optimization (2 hours)
1. Analyze query performance with EXPLAIN
2. Add appropriate indexes for common queries
3. Optimize the analytics views
4. Implement query caching strategies

### Task 4: Security Implementation (1.5 hours)
1. Create role-based access control
2. Implement data encryption for sensitive fields
3. Add audit logging for critical operations
4. Test security measures

### Task 5: Analytics and Reporting (1.5 hours)
1. Create comprehensive reporting views
2. Implement real-time dashboard queries
3. Build customer segmentation logic
4. Create inventory management reports

## 5. Testing and Validation

### 5.1 Data Integrity Tests
```sql
-- Test referential integrity
INSERT INTO order_items (order_id, product_id, quantity, unit_price, total_price)
VALUES (99999, 1, 1, 10.00, 10.00); -- Should fail

-- Test constraints
INSERT INTO products (sku, name, category_id, base_price)
VALUES ('', 'Test Product', 1, -10.00); -- Should fail

-- Test business rules
CALL AddToCart(1, NULL, 1, NULL, 1000, @success, @message);
SELECT @success, @message; -- Should fail if insufficient inventory
```

### 5.2 Performance Tests
```sql
-- Test query performance
EXPLAIN SELECT * FROM products WHERE category_id = 1 AND is_active = TRUE;

-- Test with large datasets
SELECT COUNT(*) FROM orders WHERE created_at >= '2024-01-01';

-- Test analytics queries
SELECT * FROM sales_analytics WHERE order_date >= '2024-01-01';
```

### 5.3 Business Logic Tests
```sql
-- Test order processing
CALL ProcessOrder(1, 1, '{"address": "123 Main St"}', '{"address": "456 Oak Ave"}', 
                  'credit_card', 'standard', 'SAVE10', @order_id, @success, @message);
SELECT @order_id, @success, @message;

-- Verify inventory reservation
SELECT * FROM inventory WHERE product_id = 1;
```

## 6. Deliverables

1. **Complete SQL schema** with all tables, indexes, and constraints
2. **Stored procedures** for core business operations
3. **Analytics views** and reporting queries
4. **Performance analysis** with optimization recommendations
5. **Security implementation** with access controls
6. **Test results** demonstrating system functionality
7. **Documentation** explaining design decisions and trade-offs

## 7. Evaluation Criteria

### Excellent (90-100%)
- All requirements implemented correctly
- Optimal database design with proper normalization
- Efficient queries and appropriate indexing
- Comprehensive error handling
- Security best practices implemented
- Thorough testing and documentation

### Good (80-89%)
- Most requirements implemented
- Good database design with minor issues
- Generally efficient queries
- Basic error handling
- Some security measures
- Adequate testing

### Satisfactory (70-79%)
- Core requirements implemented
- Functional database design
- Working queries with room for optimization
- Limited error handling
- Basic security
- Minimal testing

## 8. Extension Challenges

For advanced students, consider these additional features:

1. **Multi-warehouse inventory** management
2. **Advanced pricing rules** with customer-specific pricing
3. **Subscription products** with recurring billing
4. **Product reviews and ratings** system
5. **Advanced analytics** with machine learning insights
6. **API integration** for payment processing
7. **Real-time notifications** for inventory alerts
8. **Multi-currency support** with exchange rates

---

**Navigation**

[← Previous: Integration and Migration](10-03-integration-migration.md) | [Next → Data Warehouse Implementation Project](project-02-data-warehouse.md)

_Last updated: 2025-01-21_ 