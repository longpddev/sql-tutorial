# Project 3: High-Availability System

> **Assessment Project • Operations Focus**  
> Estimated time: 6-8 hours | Difficulty: ★★★★★

## 1. Project Overview

Design and implement a high-availability MySQL system with replication, failover, monitoring, and disaster recovery. Build a production-ready infrastructure that maintains 99.9% uptime with automated recovery procedures.

**Learning Objectives:**
- Configure master-slave replication
- Implement automated failover mechanisms
- Set up comprehensive monitoring and alerting
- Design disaster recovery procedures
- Handle operational challenges

> **Prerequisites:** Complete [Replication and High Availability](07-03-replication-high-availability.md) and [Monitoring and Maintenance](09-03-monitoring-maintenance.md).

## 2. System Requirements

### 2.1 Availability Targets
- **Uptime**: 99.9% (8.76 hours downtime/year)
- **RTO**: Recovery Time Objective < 5 minutes
- **RPO**: Recovery Point Objective < 1 minute data loss
- **Failover**: Automated detection and recovery
- **Monitoring**: 24/7 system health monitoring

### 2.2 Architecture Components
- Primary MySQL server (master)
- Secondary MySQL servers (slaves/replicas)
- Load balancer for read distribution
- Monitoring and alerting system
- Backup and recovery infrastructure
- Failover automation scripts

## 3. High Availability Setup

### 3.1 Master-Slave Replication Configuration

```sql
-- Master server configuration (my.cnf)
-- [mysqld]
-- server-id = 1
-- log-bin = mysql-bin
-- binlog-format = ROW
-- sync_binlog = 1
-- innodb_flush_log_at_trx_commit = 1
-- gtid_mode = ON
-- enforce_gtid_consistency = ON

-- Create replication user on master
CREATE USER 'replication'@'%' IDENTIFIED BY 'strong_replication_password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;

-- Show master status
SHOW MASTER STATUS;

-- Slave server configuration (my.cnf)
-- [mysqld]
-- server-id = 2
-- relay-log = relay-bin
-- read_only = 1
-- gtid_mode = ON
-- enforce_gtid_consistency = ON

-- Configure slave replication
CHANGE MASTER TO
    MASTER_HOST = 'master_server_ip',
    MASTER_USER = 'replication',
    MASTER_PASSWORD = 'strong_replication_password',
    MASTER_AUTO_POSITION = 1;

START SLAVE;
SHOW SLAVE STATUS\G
```

### 3.2 Multi-Master Setup for High Availability

```sql
-- Server A configuration
-- server-id = 1
-- auto_increment_increment = 2
-- auto_increment_offset = 1
-- log-slave-updates = 1

-- Server B configuration  
-- server-id = 2
-- auto_increment_increment = 2
-- auto_increment_offset = 2
-- log-slave-updates = 1

-- Configure bidirectional replication
-- On Server A:
CHANGE MASTER TO
    MASTER_HOST = 'server_b_ip',
    MASTER_USER = 'replication',
    MASTER_PASSWORD = 'password',
    MASTER_AUTO_POSITION = 1;

-- On Server B:
CHANGE MASTER TO
    MASTER_HOST = 'server_a_ip',
    MASTER_USER = 'replication',
    MASTER_PASSWORD = 'password',
    MASTER_AUTO_POSITION = 1;
```

### 3.3 Monitoring and Health Checks

```sql
-- Create monitoring database
CREATE DATABASE monitoring;
USE monitoring;

-- Health check table
CREATE TABLE health_checks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    server_name VARCHAR(50) NOT NULL,
    check_type ENUM('replication', 'disk_space', 'connections', 'performance') NOT NULL,
    status ENUM('healthy', 'warning', 'critical') NOT NULL,
    message TEXT,
    check_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_server_time (server_name, check_time),
    INDEX idx_status (status)
);

-- Replication monitoring procedure
DELIMITER //
CREATE PROCEDURE CheckReplicationHealth()
BEGIN
    DECLARE v_slave_running VARCHAR(10);
    DECLARE v_seconds_behind INT;
    DECLARE v_status VARCHAR(20) DEFAULT 'healthy';
    DECLARE v_message TEXT DEFAULT '';
    
    -- Check slave status
    SELECT 
        Slave_SQL_Running,
        Seconds_Behind_Master
    INTO v_slave_running, v_seconds_behind
    FROM (SHOW SLAVE STATUS) AS slave_status;
    
    -- Determine health status
    IF v_slave_running != 'Yes' THEN
        SET v_status = 'critical';
        SET v_message = 'Slave SQL thread not running';
    ELSEIF v_seconds_behind > 60 THEN
        SET v_status = 'warning';
        SET v_message = CONCAT('Replication lag: ', v_seconds_behind, ' seconds');
    ELSEIF v_seconds_behind > 300 THEN
        SET v_status = 'critical';
        SET v_message = CONCAT('Critical replication lag: ', v_seconds_behind, ' seconds');
    ELSE
        SET v_message = CONCAT('Replication healthy, lag: ', COALESCE(v_seconds_behind, 0), ' seconds');
    END IF;
    
    -- Record health check
    INSERT INTO health_checks (server_name, check_type, status, message)
    VALUES (@@hostname, 'replication', v_status, v_message);
END //

-- Performance monitoring procedure
CREATE PROCEDURE CheckPerformanceHealth()
BEGIN
    DECLARE v_threads_connected INT;
    DECLARE v_threads_running INT;
    DECLARE v_status VARCHAR(20) DEFAULT 'healthy';
    DECLARE v_message TEXT;
    
    -- Get connection metrics
    SELECT VARIABLE_VALUE INTO v_threads_connected
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Threads_connected';
    
    SELECT VARIABLE_VALUE INTO v_threads_running
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Threads_running';
    
    -- Check thresholds
    IF v_threads_connected > 100 THEN
        SET v_status = 'warning';
        SET v_message = CONCAT('High connection count: ', v_threads_connected);
    END IF;
    
    IF v_threads_running > 50 THEN
        SET v_status = 'critical';
        SET v_message = CONCAT(v_message, ' High running threads: ', v_threads_running);
    END IF;
    
    IF v_status = 'healthy' THEN
        SET v_message = CONCAT('Performance healthy - Connections: ', v_threads_connected, ', Running: ', v_threads_running);
    END IF;
    
    INSERT INTO health_checks (server_name, check_type, status, message)
    VALUES (@@hostname, 'performance', v_status, v_message);
END //

DELIMITER ;

-- Schedule health checks
CREATE EVENT health_check_replication
ON SCHEDULE EVERY 30 SECOND
DO CALL CheckReplicationHealth();

CREATE EVENT health_check_performance  
ON SCHEDULE EVERY 1 MINUTE
DO CALL CheckPerformanceHealth();
```

### 3.4 Automated Failover Script

```bash
#!/bin/bash
# failover.sh - Automated MySQL failover script

MASTER_HOST="10.0.1.10"
SLAVE_HOST="10.0.1.11"
VIP="10.0.1.100"  # Virtual IP for applications
MYSQL_USER="monitor"
MYSQL_PASS="monitor_password"

# Function to check MySQL connectivity
check_mysql() {
    local host=$1
    mysql -h $host -u $MYSQL_USER -p$MYSQL_PASS -e "SELECT 1" &>/dev/null
    return $?
}

# Function to promote slave to master
promote_slave() {
    echo "Promoting slave $SLAVE_HOST to master..."
    
    # Stop slave processes
    mysql -h $SLAVE_HOST -u $MYSQL_USER -p$MYSQL_PASS -e "STOP SLAVE;"
    
    # Reset slave configuration
    mysql -h $SLAVE_HOST -u $MYSQL_USER -p$MYSQL_PASS -e "RESET SLAVE ALL;"
    
    # Enable writes on new master
    mysql -h $SLAVE_HOST -u $MYSQL_USER -p$MYSQL_PASS -e "SET GLOBAL read_only = 0;"
    
    # Move VIP to new master
    # This would typically involve updating load balancer configuration
    # or moving a virtual IP address
    
    echo "Failover completed. New master: $SLAVE_HOST"
    
    # Send alert notification
    send_alert "CRITICAL: MySQL failover completed. New master: $SLAVE_HOST"
}

# Function to send alerts
send_alert() {
    local message="$1"
    # Send email, Slack notification, etc.
    echo "$message" | mail -s "MySQL Failover Alert" admin@company.com
}

# Main failover logic
main() {
    echo "Checking master health..."
    
    if ! check_mysql $MASTER_HOST; then
        echo "Master $MASTER_HOST is down. Checking slave..."
        
        if check_mysql $SLAVE_HOST; then
            echo "Slave $SLAVE_HOST is healthy. Initiating failover..."
            promote_slave
        else
            echo "CRITICAL: Both master and slave are down!"
            send_alert "CRITICAL: Both MySQL servers are down!"
            exit 1
        fi
    else
        echo "Master $MASTER_HOST is healthy."
    fi
}

# Run health check every 30 seconds
while true; do
    main
    sleep 30
done
```

### 3.5 Backup and Recovery Strategy

```sql
-- Backup procedures
DELIMITER //

CREATE PROCEDURE CreateFullBackup()
BEGIN
    DECLARE backup_file VARCHAR(255);
    DECLARE backup_command TEXT;
    
    SET backup_file = CONCAT('/backups/full_backup_', DATE_FORMAT(NOW(), '%Y%m%d_%H%i%s'), '.sql');
    SET backup_command = CONCAT('mysqldump --single-transaction --routines --triggers --all-databases > ', backup_file);
    
    -- Log backup start
    INSERT INTO monitoring.backup_log (backup_type, status, start_time, filename)
    VALUES ('full', 'started', NOW(), backup_file);
    
    -- Execute backup (this would be done externally)
    -- System command execution would happen outside MySQL
    
    -- Log backup completion (would be updated by external script)
    -- UPDATE monitoring.backup_log SET status = 'completed', end_time = NOW() WHERE filename = backup_file;
END //

CREATE PROCEDURE CreateIncrementalBackup()
BEGIN
    DECLARE last_binlog VARCHAR(255);
    DECLARE backup_dir VARCHAR(255) DEFAULT '/backups/incremental/';
    
    -- Get current binary log position
    SELECT File INTO last_binlog FROM (SHOW MASTER STATUS) AS ms;
    
    -- Copy binary logs for incremental backup
    -- This would typically be done with external tools like mysqlbinlog
    
    INSERT INTO monitoring.backup_log (backup_type, status, start_time, filename)
    VALUES ('incremental', 'completed', NOW(), CONCAT(backup_dir, last_binlog));
END //

DELIMITER ;

-- Backup logging table
CREATE TABLE monitoring.backup_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    backup_type ENUM('full', 'incremental') NOT NULL,
    status ENUM('started', 'completed', 'failed') NOT NULL,
    start_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    end_time TIMESTAMP NULL,
    filename VARCHAR(500),
    file_size BIGINT,
    error_message TEXT,
    INDEX idx_type_time (backup_type, start_time)
);

-- Schedule backups
CREATE EVENT daily_full_backup
ON SCHEDULE EVERY 1 DAY STARTS '2024-01-01 02:00:00'
DO CALL CreateFullBackup();

CREATE EVENT hourly_incremental_backup
ON SCHEDULE EVERY 1 HOUR
DO CALL CreateIncrementalBackup();
```

## 4. Implementation Tasks

### Task 1: Replication Setup (2 hours)
- Configure master-slave replication
- Test replication with sample data
- Verify GTID-based replication

### Task 2: Monitoring Implementation (2 hours)
- Create health check procedures
- Set up automated monitoring
- Configure alerting mechanisms

### Task 3: Failover Automation (2 hours)
- Implement failover detection
- Create promotion procedures
- Test failover scenarios

### Task 4: Backup Strategy (1.5 hours)
- Set up automated backups
- Test recovery procedures
- Document recovery processes

### Task 5: Load Testing (1 hour)
- Test system under load
- Verify failover during traffic
- Measure recovery times

## 5. Testing Scenarios

### 5.1 Planned Failover Test
```sql
-- Simulate planned maintenance
-- 1. Stop writes to master
SET GLOBAL read_only = 1;

-- 2. Wait for slave to catch up
SHOW SLAVE STATUS\G

-- 3. Promote slave to master
-- 4. Redirect application traffic
-- 5. Verify system functionality
```

### 5.2 Unplanned Failover Test
```bash
# Simulate master failure
sudo systemctl stop mysql  # On master server

# Monitor failover automation
tail -f /var/log/mysql-failover.log

# Verify application connectivity
mysql -h $VIP -u app_user -p -e "SELECT NOW();"
```

## 6. Deliverables

1. **Complete HA setup** with master-slave replication
2. **Monitoring system** with health checks and alerting
3. **Automated failover** scripts and procedures
4. **Backup and recovery** strategy with testing
5. **Load testing results** and performance metrics
6. **Operational runbook** for maintenance procedures
7. **Disaster recovery plan** with RTO/RPO targets

## 7. Evaluation Criteria

- **Architecture (25%)**: Proper HA design and implementation
- **Automation (25%)**: Effective failover and monitoring automation
- **Reliability (20%)**: System meets availability targets
- **Recovery (15%)**: Backup and disaster recovery procedures
- **Documentation (15%)**: Comprehensive operational procedures

## 8. Advanced Extensions

- **Multi-region setup** with geographic distribution
- **Database clustering** with MySQL Cluster (NDB)
- **Container orchestration** with Kubernetes
- **Cloud integration** with managed services
- **Advanced monitoring** with Prometheus/Grafana

---

**Navigation**

[← Previous: Data Warehouse Implementation Project](project-02-data-warehouse.md)

_Last updated: 2025-01-21_ 