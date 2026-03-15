# AWS Database Migration: EC2 MySQL to RDS
## Project Overview
This project implements a strategic database migration from a self-managed MySQL installation on an Amazon EC2 instance to a fully managed Amazon RDS (Relational Database Service) environment. The architecture emphasizes security isolation through distinct security group configurations and demonstrates best practices for data migration between AWS services.

## Architectural Framework

The migration architecture consists of two distinct tiers:

## Technical Implementation Details
### Security Group Configuration

#### Web Security Group (WebSG)
| Protocol | Port | Source | Purpose |
|:---------|:-----|:-------|:--------|
| TCP | 22 | 0.0.0.0/0 | SSH administrative access |
| TCP | 80 | 0.0.0.0/0 | HTTP web traffic |

#### Database Security Group (DBSG)
| Protocol | Port | Source | Purpose |
|:---------|:-----|:-------|:--------|
| TCP | 3306 | WebSG ID | MySQL database access restricted to web tier |

Security Architecture Note: The DBSG references the WebSG by its security group ID rather than IP CIDR blocks. This creates a dynamic security boundary that automatically includes any EC2 instances associated with WebSG, eliminating the need for manual IP updates and reducing configuration drift.

## Source Environment Configuration
An Ubuntu EC2 instance was provisioned with the WebSG attached. MySQL Server was installed and configured to serve as the source database:
```
# System update and MySQL installation
sudo apt update
sudo apt install mysql-server -y

# Service verification
sudo systemctl status mysql

# Security hardening
sudo mysql_secure_installation
```
### Database Initialization:
```
CREATE DATABASE application_db;
USE application_db;

CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_sessions (
    session_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    login_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

INSERT INTO users (username, email) VALUES
('admin_user', 'admin@example.com'),
('app_user', 'user@example.com');
```
## Target Environment: RDS Provisioning
An RDS for MySQL instance was created with the following specifications:

Database Engine: **MySQL 8.4.7**

Instance Class: **db.t4g.micro**

Storage Allocation: **20GB General Purpose SSD**

Network Configuration: **Default VPC with multi-AZ deployment**

Public Accessibility: **Enabled (temporary for migration)**

Security Group Assignment: **DBSG**

Initial Database: Pre-created as **'application_db'**

### Data Migration Process
The migration utilized MySQL's native dump and restore utilities. A critical implementation detail emerged during this phase that required troubleshooting.

Phase 1: Source Database Export
```
# Export complete database including schema and data
mysqldump -u root -p application_db > application_db_backup.sql

# Verify export integrity
tail -20 application_db_backup.sql
wc -l application_db_backup.sql
```
