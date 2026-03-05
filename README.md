# ec2-to-rds-data-migration
AWS Database Migration: EC2 (Self-Managed) to RDS (Managed)
## 📌 Project Overview
This project demonstrates the migration of a MySQL database from a standalone Amazon EC2 instance to a fully managed Amazon RDS environment. The primary focus is on implementing a two-tier architecture with strict security isolation using nested Security Groups.

## 🏗️ Architecture Design
The infrastructure is designed with a "Security-First" approach:

Public Subnet: Hosts the EC2 Instance (Web/App Tier).

Private Subnet: Hosts the Amazon RDS Instance (Data Tier).

Security Isolation: The RDS instance is configured to only accept inbound traffic on Port 3306 from the WebSG (Security Group of the EC2), preventing direct exposure to the public internet.

## 🛠️ Tech Stack
Cloud Provider: AWS (EC2, RDS, VPC, Security Groups)

Database: MySQL / MariaDB

Tooling: mysqldump, MySQL Client, Lucidchart (Documentation)

## 🚀 Step-by-Step Execution
1. Security Group Setup
WebSG (EC2): Allowed Inbound Port 22 (SSH) and Port 80 (HTTP).

DBSG (RDS): Allowed Inbound Port 3306 (MySQL) but restricted the source to the WebSG ID.

2. Source Database Preparation (EC2)
On the EC2 instance, I installed the MySQL server and seeded it with initial data:
```
Bash
# Install MySQL Client/Server
sudo yum install mariadb-server -y
sudo systemctl start mariadb
```
```
# Create Database and Table
mysql -u root -p
CREATE DATABASE sales_db;
USE sales_db;
CREATE TABLE users (id INT, name VARCHAR(50));
INSERT INTO users VALUES (1, 'Nandini');
```
## 3. Data Migration (The Cutover)
I performed a logical backup of the database and migrated it to the RDS endpoint:
```
Bash
# Step A: Export data from EC2
mysqldump -u root -p sales_db > backup.sql

# Step B: Import data to RDS
mysql -h <rds-endpoint-url> -P 3306 -u <admin-user> -p sales_db < backup.sql
```
## 4. Validation
Connected to the RDS instance from the EC2 terminal to verify data integrity:
```
Bash
mysql -h <rds-endpoint-url> -P 3306 -u <admin-user> -p
SHOW DATABASES;
SELECT * FROM users;
```
## 💡 Key Learnings
Managed vs. Self-Managed: Transitioning to RDS reduces operational overhead (backups, patching).

Security Group Nesting: Learned how to reference one SG inside another to create a "Zero Trust" internal network.

Data Portability: Practiced using standard CLI tools (mysqldump) for cloud-to-cloud migrations.
