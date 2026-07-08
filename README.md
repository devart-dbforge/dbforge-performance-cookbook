# dbForge Performance Cookbook: Practical Database Performance Tuning Examples

This repository contains practical, copy-paste-ready examples showing how to diagnose and fix common database performance problems with dbForge solutions. It covers real-world database performance tuning scenarios such as slow queries, execution plan analysis, indexing improvements, query pattern optimization, and monitoring workflows across major database platforms.

## Purpose of this repository

The goal of this repository is to provide a central collection of working examples that help developers, DBAs, and data teams solve real database performance problems faster. It is designed as a hands-on cookbook for learning how to investigate bottlenecks, apply tuning changes, and validate improvements with dbForge tools.

## Who this repository is for

This repository is intended for:

- Database developers investigating SQL performance issues
- Software developers exploring ways to improve interactions between applications and databases
- DBAs looking for optimization opportunities
- Backend engineers focused on achieving high scalability and reducing request latency
- DevOps and platform teams monitoring database performance after deployments and infrastructure changes
- Technical users responsible for SQL performance and database health

## Why these performance examples are useful

The examples in this repository help you:

- Investigate slow queries with practical tuning scenarios
- Understand common performance issues by analyzing the before and after results
- Improve query design, indexing, and execution plans
- Learn how to use dbForge tools such as Query Profiler and MySQL Profiler in real SQL troubleshooting workflows
- Apply proven database performance tuning techniques to production-like cases

## What you will find in this repository

The repository is organized by performance issue. Each directory contains focused examples and the corresponding README files explaining a specific issue, why it happens, how to detect it, and how to fix it.

The main topics include slow query troubleshooting, execution plan review, indexing strategy, query optimization patterns, and performance monitoring.

## Prerequisites

The examples in this repository show performance optimization methods based on the metrics provided by Query Profiler, an integrated tool available in dbForge Studio for SQL Server. To use the examples, [download](https://www.devart.com/dbforge/sql/studio/download.html), install, and open the Studio.

These examples use AdventureWorks2025, a sample SQL Server database. To reproduce the example scenarios, connect to AdventureWorks2025 in dbForge Studio for SQL Server and run the following script to create sample tables and populate them with data:

<details>
<summary>Show script</summary>

```sql
DROP TABLE IF EXISTS sales.big_orders;
GO
 
CREATE TABLE sales.big_orders (
order_id BIGINT IDENTITY PRIMARY KEY,
customer_id INT NOT NULL,
salesperson_id INT NOT NULL,
territory_id INT NOT NULL,
order_date DATETIME NOT NULL,
ship_date DATETIME NULL,
total_amount MONEY NOT NULL,
tax_amount MONEY NOT NULL,
freight MONEY NOT NULL,
order_status CHAR(1) NOT NULL,
payment_method VARCHAR(20) NOT NULL,
comments NVARCHAR(400) NULL
);
GO
 
;WITH numbers AS
(
SELECT TOP (5000000)
ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
FROM sys.all_objects a
CROSS JOIN sys.all_objects b
CROSS JOIN sys.all_objects c
)
INSERT INTO sales.big_orders
(
customer_id,
salesperson_id,
territory_id,
order_date,
ship_date,
total_amount,
tax_amount,
freight,
order_status,
payment_method,
comments
)
SELECT
CASE
-- skewed distribution for parameter sniffing demos
WHEN n % 100 < 40 THEN 1
WHEN n % 100 < 60 THEN 2
ELSE ABS(CHECKSUM(NEWID())) % 50000 + 1
END,
ABS(CHECKSUM(NEWID())) % 500 + 1,
ABS(CHECKSUM(NEWID())) % 20 + 1,
DATEADD
(
DAY,
-(ABS(CHECKSUM(NEWID())) % 3650),
GETDATE()
),
DATEADD
(
DAY,
ABS(CHECKSUM(NEWID())) % 10,
DATEADD
(
DAY,
-(ABS(CHECKSUM(NEWID())) % 3650),
GETDATE()
)
),
CAST((ABS(CHECKSUM(NEWID())) % 500000) / 10.0 + 50 AS MONEY),
CAST((ABS(CHECKSUM(NEWID())) % 50000) / 10.0 AS MONEY),
CAST((ABS(CHECKSUM(NEWID())) % 20000) / 10.0 AS MONEY),
CASE
WHEN n % 100 < 70 THEN 'S' -- shipped
WHEN n % 100 < 85 THEN 'P' -- pending
WHEN n % 100 < 95 THEN 'C' -- completed
ELSE 'X' -- cancelled
END,
CASE
WHEN n % 4 = 0 THEN 'Credit Card'
WHEN n % 4 = 1 THEN 'PayPal'
WHEN n % 4 = 2 THEN 'Wire Transfer'
ELSE 'Cash'
END,
CASE
WHEN n % 50 = 0 THEN 'Urgent order'
WHEN n % 75 = 0 THEN 'Customer requested priority shipping'
ELSE NULL
END
FROM numbers;
GO
 
DROP TABLE IF EXISTS production.big_products;
GO
 
CREATE TABLE production.big_products (
product_id INT IDENTITY PRIMARY KEY,
product_name NVARCHAR(200) NOT NULL,
category_id INT NOT NULL,
supplier_id INT NOT NULL,
price MONEY NOT NULL,
cost MONEY NOT NULL,
stock_quantity INT NOT NULL,
reorder_level INT NOT NULL,
is_active BIT NOT NULL,
created_date DATETIME NOT NULL,
modified_date DATETIME NOT NULL
);
GO
 
;WITH numbers AS
(
SELECT TOP (2000000)
ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
FROM sys.all_objects a
CROSS JOIN sys.all_objects b
CROSS JOIN sys.all_objects c
)
INSERT INTO production.big_products
(
product_name,
category_id,
supplier_id,
price,
cost,
stock_quantity,
reorder_level,
is_active,
created_date,
modified_date
)
SELECT
CONCAT
(
CASE n % 10
WHEN 0 THEN 'Laptop '
WHEN 1 THEN 'Phone '
WHEN 2 THEN 'Monitor '
WHEN 3 THEN 'Keyboard '
WHEN 4 THEN 'Mouse '
WHEN 5 THEN 'Tablet '
WHEN 6 THEN 'Printer '
WHEN 7 THEN 'Camera '
WHEN 8 THEN 'Speaker '
ELSE 'Accessory '
END,
n
),
ABS(CHECKSUM(NEWID())) % 50 + 1,
ABS(CHECKSUM(NEWID())) % 1000 + 1,
CAST((ABS(CHECKSUM(NEWID())) % 200000) / 10.0 + 5 AS MONEY),
CAST((ABS(CHECKSUM(NEWID())) % 100000) / 10.0 + 1 AS MONEY),
ABS(CHECKSUM(NEWID())) % 5000,
ABS(CHECKSUM(NEWID())) % 100,
CASE
WHEN n % 100 < 90 THEN 1
ELSE 0
END,
DATEADD
(
DAY,
-(ABS(CHECKSUM(NEWID())) % 3650),
GETDATE()
),
GETDATE()
FROM numbers;
GO
 
DROP TABLE IF EXISTS person.big_person;
GO
 
CREATE TABLE person.big_person (
person_id INT IDENTITY PRIMARY KEY,
first_name NVARCHAR(100) NOT NULL,
last_name NVARCHAR(100) NOT NULL,
city NVARCHAR(100) NOT NULL,
country NVARCHAR(100) NOT NULL,
email_address VARCHAR(200) NOT NULL,
phone_number VARCHAR(20) NOT NULL,
birth_date DATE NOT NULL,
created_date DATETIME NOT NULL
);
GO
 
;WITH numbers AS
(
SELECT TOP (2000000)
ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
FROM sys.all_objects a
CROSS JOIN sys.all_objects b
CROSS JOIN sys.all_objects c
)
INSERT INTO person.big_person
(
first_name,
last_name,
city,
country,
email_address,
phone_number,
birth_date,
created_date
)
SELECT
CONCAT
(
CASE n % 10
WHEN 0 THEN 'John'
WHEN 1 THEN 'Michael'
WHEN 2 THEN 'Sarah'
WHEN 3 THEN 'David'
WHEN 4 THEN 'Emily'
WHEN 5 THEN 'Daniel'
WHEN 6 THEN 'Olivia'
WHEN 7 THEN 'James'
WHEN 8 THEN 'Sophia'
ELSE 'Robert'
END,
n
),
CONCAT
(
CASE n % 10
WHEN 0 THEN 'Smith'
WHEN 1 THEN 'Johnson'
WHEN 2 THEN 'Williams'
WHEN 3 THEN 'Brown'
WHEN 4 THEN 'Jones'
WHEN 5 THEN 'Garcia'
WHEN 6 THEN 'Miller'
WHEN 7 THEN 'Davis'
WHEN 8 THEN 'Wilson'
ELSE 'Taylor'
END,
n
),
CONCAT('City ', ABS(CHECKSUM(NEWID())) % 500),
CASE
WHEN n % 5 = 0 THEN 'USA'
WHEN n % 5 = 1 THEN 'Canada'
WHEN n % 5 = 2 THEN 'Germany'
WHEN n % 5 = 3 THEN 'France'
ELSE 'UK'
END,
CONCAT('user', n, '@demo.com'),
CONCAT
(
'+1-',
RIGHT('000' + CAST(ABS(CHECKSUM(NEWID())) % 999 AS VARCHAR(3)), 3),
'-',
RIGHT('000' + CAST(ABS(CHECKSUM(NEWID())) % 999 AS VARCHAR(3)), 3),
'-',
RIGHT('0000' + CAST(ABS(CHECKSUM(NEWID())) % 9999 AS VARCHAR(4)), 4)
),
DATEADD
(
DAY,
-(ABS(CHECKSUM(NEWID())) % 20000),
GETDATE()
),
DATEADD
(
DAY,
-(ABS(CHECKSUM(NEWID())) % 3650),
GETDATE()
)
FROM numbers;
GO
```

</details>

> After each scenario, restore the database to its default state.

## Related products

The tutorials in this repository cover the following dbForge products:

- [dbForge Studio for SQL Server](https://www.devart.com/dbforge/sql/studio/)
- [dbForge Studio for MySQL](https://www.devart.com/dbforge/mysql/studio/)
- [dbForge Studio for PostgreSQL](https://www.devart.com/dbforge/postgresql/studio/)
- [dbForge Studio for Oracle](https://www.devart.com/dbforge/oracle/studio/)
- [dbForge Edge](https://www.devart.com/dbforge/edge/) (cross-database workflows)

**Databases and tools:**

- SQL Server
- MySQL & MariaDB
- PostgreSQL
- Oracle
- Azure DevOps, Jenkins, GitLab CI/CD

## Learn more

- [Devart Academy](https://www.devart.com/academy/): Free structured courses and guided materials on databases and dbForge solutions
- [Devart YouTube channel](https://www.youtube.com/DevartSoftware): Video tutorials, feature overviews, and practical demos of dbForge
