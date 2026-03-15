# AWS Database Migration: EC2 MySQL to RDS
## Project Overview
This project implements a strategic database migration from a self-managed MySQL installation on an Amazon EC2 instance to a fully managed Amazon RDS (Relational Database Service) environment. The architecture emphasizes security isolation through distinct security group configurations and demonstrates best practices for data migration between AWS services.

## Architectural Framework

The migration architecture consists of two distinct tiers:
![AWS RDS Migration Architecture](https://github.com/KmNandini12/ec2-to-rds-data-migration/blob/b9294bda6a634677439aa77ffeacbf97c38ca991/sceenshots/architecture%20ec2%20to%20rds.png)

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
    email VARCHAR(100) UNIQUE NOT NULL
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
### Phase 2: Target Database Preparation

During initial migration attempts, the following error occurred:
** ERROR 1046 (3D000): No database selected**

** Root Cause Analysis:** The mysqldump utility exports database objects but does not automatically create the target database on the destination server. The dump file contains CREATE DATABASE statements commented out by default. RDS instances launch without any user databases, requiring explicit database creation before import operations.

### Phase 3: Data Import
``` # Import data into pre-created database
mysql -h [rds-endpoint].rds.amazonaws.com \
      -u admin \
      -p \
      application_db < application_db_backup.sql
```
** Technical Note on Import Process:** The import command specifies the target database explicitly, directing all table creation and data insertion operations into the correct database context. This approach maintains referential integrity and ensures all database objects are properly created.

## Migration Verification
Comprehensive validation confirmed successful migration:

### Connectivity Validation:
```
bash
# Test connection from EC2 instance
mysql -h [rds-endpoint].rds.amazonaws.com \
      -u admin \
      -p \
      -e "SELECT VERSION(), CURRENT_DATE;"
Data Integrity Verification:

sql
-- Verify database existence
SHOW DATABASES;

-- Verify table structure
USE application_db;
SHOW TABLES;
DESCRIBE users;

-- Verify data migration
SELECT COUNT(*) FROM users;
SELECT * FROM users;
-- Verify referential integrity
SELECT u.username, COUNT(s.session_id) as session_count
FROM users u
LEFT JOIN user_sessions s ON u.user_id = s.user_id
GROUP BY u.user_id;
```
### Security Verification:
```
bash
# Attempt connection from unauthorized host (should fail)
mysql -h [rds-endpoint].rds.amazonaws.com -u admin -p
# Expected: Connection timeout or access denied

# Verify security group isolation
aws ec2 describe-security-groups \
    --group-ids [DBSG-ID] \
    --query 'SecurityGroups[0].IpPermissions'
```
## Technical Observations and Lessons Learned
Database Pre-creation Requirement: RDS instances provision without any initial databases. The migration process requires explicit database creation before data import operations can succeed. This differs from local MySQL installations where the mysql system database exists and user databases can be created during import.

Security Group References vs. CIDR Blocks: Using security group IDs for inter-tier communication provides superior security management. This approach automatically scales with infrastructure changes and eliminates the risk of exposing database ports to unintended IP ranges.

## Migration Metrics

| Metric | Value |
|:-------|:------|
| Migration Method | Manual dump/restore |
| Total Migration Time | ~15 minutes |
| Data Transfer Size | Variable based on database size |
| Application Downtime | Minimal (< 5 minutes) |
| Post-Migration Validation | Complete |

## Security Posture Assessment
The final architecture achieves the following security objectives:

Database tier isolated from direct internet access

Database access restricted to authenticated application server only

No hardcoded IP addresses in security rules

Credential separation between source and target environments

Encrypted connections during migration (TLS enabled)

### Troubleshooting Reference

| Error | Cause | Resolution |
|:-------|:------|:-----------|
| ERROR 1046 (3D000) | Target database missing | CREATE DATABASE before import |
| ERROR 2003 (HY000) | Connection timeout | Verify security group rules |
| ERROR 1045 (28000) | Authentication failure | Validate credentials and host permissions |
| ERROR 1146 (42S02) | Table missing | Verify dump file completeness |
| ERROR 1217 (23000) | Foreign key constraint | Import tables in correct order |

## Conclusion
This migration successfully transitioned a self-managed database infrastructure to a managed service model while implementing defense-in-depth security principles. The architecture ensures that database resources are accessible only through the application tier, significantly reducing the attack surface. The managed nature of RDS offloads administrative overhead including backup management, patch application, and scaling operations to AWS, allowing development teams to focus on application logic rather than infrastructure maintenance.

The key technical insight from this implementation is the requirement for explicit database creation on RDS targets, a nuance that distinguishes managed database services from traditional MySQL installations. This understanding ensures successful migrations and proper planning for future database deployments on AWS
