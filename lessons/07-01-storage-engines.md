# Storage Engines

> **Module 7 • Lesson 1**  
> Estimated time: 30 min | Difficulty: ★★★☆☆

## 1. Why this matters

Storage engines are the foundation of how MySQL stores, retrieves, and manages your data. Each engine has different strengths, weaknesses, and use cases. Choosing the right storage engine can dramatically impact your application's performance, reliability, and feature availability. Understanding storage engines helps you make informed decisions about data architecture, from simple web applications to complex enterprise systems requiring transactions, full-text search, or in-memory processing.

> **Need a refresher?** This lesson builds on concepts from [Database Management Systems Deep Dive](01-02-dbms-deep-dive.md).

## 2. Key Concepts

- **Storage engine architecture**: Pluggable storage layer in MySQL
- **InnoDB**: Default ACID-compliant engine with row-level locking
- **MyISAM**: Legacy engine optimized for read-heavy workloads
- **Memory**: In-memory storage for temporary data and caching
- **Archive**: Compressed storage for historical data
- **Engine selection criteria**: Performance, features, and use case requirements
- **Engine-specific features**: Transactions, foreign keys, full-text search

```mermaid
graph TD
    A["MySQL Server Layer"] --> B["Storage Engine API"]
    B --> C["InnoDB<br/>ACID, Row Locks<br/>Default Choice"]
    B --> D["MyISAM<br/>Table Locks<br/>Fast Reads"]
    B --> E["Memory<br/>RAM Storage<br/>Temporary Data"]
    B --> F["Archive<br/>Compressed<br/>Historical Data"]
    B --> G["CSV<br/>Plain Text<br/>Data Exchange"]
```

## 3. Deep Dive

### 3.1 MySQL Storage Engine Architecture

MySQL uses a pluggable storage engine architecture that separates the SQL layer from the storage layer:

**Server Layer Responsibilities:**
- SQL parsing and optimization
- Query cache management
- User authentication and authorization
- Backup and replication coordination

**Storage Engine Responsibilities:**
- Data storage and retrieval
- Index management
- Transaction handling
- Lock management
- Crash recovery

```sql
-- View available storage engines
SHOW ENGINES;

-- Check default storage engine
SHOW VARIABLES LIKE 'default_storage_engine';

-- View table's storage engine
SHOW TABLE STATUS LIKE 'table_name';
```

### 3.2 InnoDB: The Default Choice

InnoDB is MySQL's default storage engine since version 5.5, and for good reason:

**Key Features:**
- **ACID Compliance**: Full transaction support
- **Row-Level Locking**: Better concurrency than table-level locking
- **Foreign Key Constraints**: Referential integrity enforcement
- **Crash Recovery**: Automatic recovery using redo/undo logs
- **MVCC**: Multi-version concurrency control
- **Clustered Indexes**: Data stored with primary key

**InnoDB Architecture:**
```sql
-- Create InnoDB table (default)
CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- InnoDB-specific features
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    total DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
        ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB;

-- Transaction support
BEGIN;
INSERT INTO customers (name, email) VALUES ('John Doe', 'john@example.com');
INSERT INTO orders (customer_id, order_date, total) 
VALUES (LAST_INSERT_ID(), CURDATE(), 299.99);
COMMIT;
```

**InnoDB Configuration:**
```sql
-- Key InnoDB settings
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';    -- Memory for caching
SHOW VARIABLES LIKE 'innodb_log_file_size';       -- Redo log size
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit'; -- Durability vs performance

-- Monitor InnoDB status
SHOW ENGINE INNODB STATUS\G
```

### 3.3 MyISAM: Legacy but Still Relevant

MyISAM was MySQL's default storage engine before InnoDB. While legacy, it still has specific use cases:

**Key Features:**
- **Table-Level Locking**: Simple but limits concurrency
- **No Transaction Support**: Not ACID compliant
- **Fast for Read-Heavy Workloads**: Optimized for SELECT operations
- **Full-Text Search**: Built-in full-text indexing (before MySQL 5.6)
- **Smaller Storage Footprint**: Less overhead than InnoDB

**When to Use MyISAM:**
- Read-only or read-mostly applications
- Data warehousing scenarios
- Temporary tables for complex queries
- Legacy applications that can't be migrated

```sql
-- Create MyISAM table
CREATE TABLE product_catalog (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200),
    description TEXT,
    category VARCHAR(50),
    price DECIMAL(10,2),
    FULLTEXT(name, description)
) ENGINE=MyISAM;

-- MyISAM full-text search
INSERT INTO product_catalog (name, description, category, price) VALUES
('Laptop Computer', 'High-performance laptop for business use', 'Electronics', 999.99),
('Office Chair', 'Ergonomic office chair with lumbar support', 'Furniture', 299.99);

-- Full-text search query
SELECT * FROM product_catalog 
WHERE MATCH(name, description) AGAINST('laptop computer' IN BOOLEAN MODE);

-- Table-level locking behavior
LOCK TABLES product_catalog WRITE;
-- All other sessions blocked from accessing this table
UPDATE product_catalog SET price = 899.99 WHERE id = 1;
UNLOCK TABLES;
```

### 3.4 Memory Engine: RAM-Based Storage

The Memory engine stores all data in RAM, providing extremely fast access:

**Key Features:**
- **In-Memory Storage**: All data stored in RAM
- **Fixed-Length Rows**: No VARCHAR, TEXT, or BLOB support
- **Hash Indexes**: Default index type for fast lookups
- **Data Lost on Restart**: Not persistent across server restarts
- **Table-Level Locking**: Similar to MyISAM

**Use Cases:**
- Session storage
- Temporary lookup tables
- Caching frequently accessed data
- Real-time analytics intermediate tables

```sql
-- Create Memory table
CREATE TABLE session_data (
    session_id CHAR(32) PRIMARY KEY,
    user_id INT NOT NULL,
    login_time TIMESTAMP,
    last_activity TIMESTAMP,
    data TEXT  -- Note: TEXT not supported, use VARCHAR instead
) ENGINE=MEMORY;

-- Correct Memory table definition
CREATE TABLE session_cache (
    session_id CHAR(32) PRIMARY KEY,
    user_id INT NOT NULL,
    login_time INT,  -- Unix timestamp
    data VARCHAR(1000)
) ENGINE=MEMORY;

-- Fast lookups
INSERT INTO session_cache VALUES 
('abc123def456', 1001, UNIX_TIMESTAMP(), 'user_preferences=dark_mode');

SELECT * FROM session_cache WHERE session_id = 'abc123def456';

-- Monitor Memory engine usage
SHOW TABLE STATUS WHERE Engine = 'MEMORY';
```

### 3.5 Archive Engine: Compressed Storage

The Archive engine is designed for storing large amounts of historical data:

**Key Features:**
- **High Compression**: Uses zlib compression
- **Insert and Select Only**: No UPDATE or DELETE operations
- **Minimal Indexes**: Only supports AUTO_INCREMENT indexes
- **Low Storage Cost**: Excellent compression ratios

**Use Cases:**
- Log data archival
- Historical transaction records
- Compliance data storage
- Data that's written once, read occasionally

```sql
-- Create Archive table
CREATE TABLE access_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT,
    request_uri VARCHAR(500),
    response_code INT
) ENGINE=Archive;

-- Insert log data (allowed)
INSERT INTO access_logs (ip_address, user_agent, request_uri, response_code)
VALUES ('192.168.1.100', 'Mozilla/5.0...', '/api/users', 200);

-- Select data (allowed)
SELECT COUNT(*) FROM access_logs WHERE response_code >= 400;

-- Update not supported
-- UPDATE access_logs SET response_code = 404 WHERE id = 1;  -- ERROR

-- Check compression ratio
SELECT 
    table_name,
    ROUND(data_length / 1024 / 1024, 2) AS data_size_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_size_mb
FROM information_schema.tables 
WHERE table_name = 'access_logs';
```

### 3.6 Other Storage Engines

**CSV Engine:**
```sql
-- CSV engine for data exchange
CREATE TABLE export_data (
    id INT,
    name VARCHAR(100),
    value DECIMAL(10,2)
) ENGINE=CSV;

-- Data stored as plain CSV file
INSERT INTO export_data VALUES (1, 'Product A', 99.99);
-- File location: datadir/database_name/export_data.CSV
```

**Federated Engine:**
```sql
-- Connect to remote MySQL server (if enabled)
CREATE TABLE remote_users (
    id INT,
    username VARCHAR(50)
) ENGINE=FEDERATED
CONNECTION='mysql://user:password@remote_host:3306/database/users';
```

## 4. Hands-On Practice

Let's compare storage engines with practical examples:

```sql
-- Create identical tables with different engines
CREATE TABLE performance_test_innodb (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE performance_test_myisam (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=MyISAM;

CREATE TABLE performance_test_memory (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(100),
    created_at INT  -- TIMESTAMP not supported
) ENGINE=Memory;

-- Insert test data
INSERT INTO performance_test_innodb (data) 
SELECT CONCAT('Test data ', n) 
FROM (
    SELECT a.N + b.N * 10 + c.N * 100 + 1 n
    FROM (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) c
) numbers 
WHERE n <= 1000;

-- Copy data to other engines
INSERT INTO performance_test_myisam (data) 
SELECT data FROM performance_test_innodb;

INSERT INTO performance_test_memory (data, created_at) 
SELECT data, UNIX_TIMESTAMP() FROM performance_test_innodb;

-- Compare table sizes and characteristics
SELECT 
    table_name,
    engine,
    table_rows,
    ROUND(data_length / 1024, 2) AS data_kb,
    ROUND(index_length / 1024, 2) AS index_kb,
    ROUND((data_length + index_length) / 1024, 2) AS total_kb
FROM information_schema.tables 
WHERE table_name LIKE 'performance_test_%';

-- Test concurrent access behavior
-- Session 1 (InnoDB - row-level locking)
BEGIN;
UPDATE performance_test_innodb SET data = 'Updated' WHERE id = 1;
-- Don't commit yet

-- Session 2 (can update different rows)
UPDATE performance_test_innodb SET data = 'Another update' WHERE id = 2;  -- Works

-- Session 1 (MyISAM - table-level locking)
UPDATE performance_test_myisam SET data = 'Updated' WHERE id = 1;

-- Session 2 (blocked until Session 1 completes)
UPDATE performance_test_myisam SET data = 'Another update' WHERE id = 2;  -- Blocks
```

**Engine Selection Decision Tree:**
```sql
-- Create a procedure to help choose storage engine
DELIMITER //
CREATE PROCEDURE recommend_storage_engine(
    IN need_transactions BOOLEAN,
    IN need_foreign_keys BOOLEAN,
    IN read_heavy BOOLEAN,
    IN need_full_text BOOLEAN,
    IN temporary_data BOOLEAN,
    IN archive_data BOOLEAN
)
BEGIN
    IF temporary_data THEN
        SELECT 'MEMORY' as recommended_engine, 
               'Fast in-memory storage for temporary data' as reason;
    ELSEIF archive_data THEN
        SELECT 'Archive' as recommended_engine,
               'Compressed storage for historical data' as reason;
    ELSEIF need_transactions OR need_foreign_keys THEN
        SELECT 'InnoDB' as recommended_engine,
               'ACID compliance and referential integrity required' as reason;
    ELSEIF read_heavy AND need_full_text THEN
        SELECT 'MyISAM' as recommended_engine,
               'Read-heavy with full-text search requirements' as reason;
    ELSE
        SELECT 'InnoDB' as recommended_engine,
               'Default choice for most applications' as reason;
    END IF;
END //
DELIMITER ;

-- Test the recommendation system
CALL recommend_storage_engine(TRUE, TRUE, FALSE, FALSE, FALSE, FALSE);
CALL recommend_storage_engine(FALSE, FALSE, TRUE, TRUE, FALSE, FALSE);
CALL recommend_storage_engine(FALSE, FALSE, FALSE, FALSE, TRUE, FALSE);
```

**Practice Exercises:**

1. **Performance Comparison**: Compare INSERT, SELECT, and UPDATE performance across engines
2. **Concurrency Testing**: Test how different engines handle concurrent access
3. **Storage Analysis**: Compare storage requirements for the same data across engines
4. **Feature Testing**: Test transactions, foreign keys, and full-text search capabilities

## 5. Common Pitfalls

### 5.1 Using Wrong Engine for Use Case
**Problem**: Choosing storage engine based on familiarity rather than requirements
```sql
-- WRONG: Using MyISAM for transactional application
CREATE TABLE financial_transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    account_id INT,
    amount DECIMAL(15,2),
    transaction_date TIMESTAMP
) ENGINE=MyISAM;  -- No transaction support!

-- CORRECT: Use InnoDB for transactional data
CREATE TABLE financial_transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    account_id INT,
    amount DECIMAL(15,2),
    transaction_date TIMESTAMP
) ENGINE=InnoDB;
```

### 5.2 Not Understanding Engine Limitations
**Problem**: Assuming all engines support the same features
```sql
-- WRONG: Trying to use foreign keys with MyISAM
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)  -- Ignored!
) ENGINE=MyISAM;

-- WRONG: Using variable-length data with Memory engine
CREATE TABLE cache_data (
    id INT PRIMARY KEY,
    data TEXT  -- Not supported by Memory engine
) ENGINE=Memory;
```

### 5.3 Ignoring Concurrency Implications
**Problem**: Not considering locking behavior impact
```sql
-- PROBLEMATIC: MyISAM in high-concurrency environment
CREATE TABLE user_sessions (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id INT,
    last_activity TIMESTAMP
) ENGINE=MyISAM;  -- Table locks will hurt concurrency
```

### 5.4 Memory Engine Data Loss
**Problem**: Not understanding Memory engine persistence
```sql
-- DANGEROUS: Critical data in Memory engine
CREATE TABLE user_preferences (
    user_id INT PRIMARY KEY,
    theme VARCHAR(20),
    language VARCHAR(10)
) ENGINE=Memory;  -- Data lost on server restart!
```

## 6. Knowledge Check

<details>
<summary>1. What are the main differences between InnoDB and MyISAM?</summary>

InnoDB supports ACID transactions, foreign keys, row-level locking, and crash recovery. MyISAM uses table-level locking, doesn't support transactions or foreign keys, but is faster for read-heavy workloads and has a smaller storage footprint.
</details>

<details>
<summary>2. When would you choose the Memory storage engine?</summary>

Use Memory engine for temporary data that needs extremely fast access, such as session storage, lookup tables, or intermediate results in complex queries. Remember that data is lost when the server restarts.
</details>

<details>
<summary>3. What are the limitations of the Archive storage engine?</summary>

Archive engine only supports INSERT and SELECT operations (no UPDATE or DELETE), has minimal indexing support (only AUTO_INCREMENT), and is designed for write-once, read-occasionally data with high compression.
</details>

<details>
<summary>4. Why is InnoDB the default storage engine in modern MySQL?</summary>

InnoDB provides ACID compliance, better concurrency through row-level locking, foreign key support, automatic crash recovery, and MVCC. These features make it suitable for most modern applications that require data integrity and concurrent access.
</details>

<details>
<summary>5. How do you determine which storage engine a table is using?</summary>

Use `SHOW TABLE STATUS LIKE 'table_name'` or query `information_schema.tables` to see the engine. You can also use `SHOW CREATE TABLE table_name` to see the complete table definition including the engine.
</details>

## 7. Further Reading

- [MySQL Storage Engine Architecture](https://dev.mysql.com/doc/refman/8.0/en/pluggable-storage-overview.html)
- [InnoDB Storage Engine](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)
- [Alternative Storage Engines](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)
- [Choosing the Right Storage Engine](https://www.percona.com/blog/2009/01/05/mysql-storage-engine-comparison/)
- [Storage Engine Performance Comparison](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)

---

**Navigation**

[← Previous: Deadlock Detection and Prevention](06-03-deadlock-detection-prevention.md) | [Next → MySQL Extensions and Functions](07-02-mysql-extensions-functions.md)

_Last updated: 2025-01-21_ 