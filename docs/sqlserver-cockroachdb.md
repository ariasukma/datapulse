# SQL Server to CockroachDB

Use this process to prepare Datapulse for data migration from SQL Server to CockroachDB.

## SQL Server

### Create Table at SQL Server

Buat database dan table di SQL Server, sebagai contoh kita akan mereplikasi 2 tables dibawah ini.

```SQL
CREATE DATABASE datapulse_demo;
GO

USE datapulse_demo;
GO

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(200),
    city VARCHAR(100),
    created_at DATETIME DEFAULT GETDATE()
);
GO

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    amount DECIMAL(12,2),
    status VARCHAR(20),
    created_at DATETIME DEFAULT GETDATE(),

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
);
GO

INSERT INTO customers
(customer_id, customer_name, email, city)
VALUES
(1,'John Doe','john@example.com','Jakarta'),
(2,'Jane Smith','jane@example.com','Bandung'),
(3,'Michael Tan','michael@example.com','Surabaya'),
(4,'Sarah Lee','sarah@example.com','Medan'),
(5,'David Wong','david@example.com','Semarang'),
(6,'Kevin Lim','kevin@example.com','Makassar'),
(7,'Lisa Chen','lisa@example.com','Yogyakarta'),
(8,'Budi Santoso','budi@example.com','Bogor'),
(9,'Andi Wijaya','andi@example.com','Bekasi'),
(10,'Siti Rahma','siti@example.com','Depok');
GO

INSERT INTO orders
(order_id, customer_id, amount, status)
VALUES
(1001,1,150.50,'PAID'),
(1002,2,200.00,'PAID'),
(1003,3,325.75,'PENDING'),
(1004,4,500.00,'PAID'),
(1005,5,125.25,'CANCELLED'),
(1006,6,890.10,'PAID'),
(1007,7,220.40,'PENDING'),
(1008,8,175.60,'PAID'),
(1009,9,999.99,'PAID'),
(1010,10,75.00,'PENDING');
GO
```

Dataset ini cukup bagus untuk menguji seluruh jalur Datapulse.

```
SQL Server -> PostgreSQL
```

Karena sudah mengandung:
Primary Key, Foreign Key, VARCHAR/STRING, DECIMAL, TIMESTAMP/DATETIME, Parent-child relationship (customers → orders).

### Enable CDC pada database

```SQL
USE datapulse_demo;
EXEC sys.sp_cdc_enable_db;
```

Enable table CDC:

```SQL
EXEC sys.sp_cdc_enable_table
    @source_schema='dbo',
    @source_name='customers',
    @role_name=NULL,
    @supports_net_changes=1;

EXEC sys.sp_cdc_enable_table
    @source_schema='dbo',
    @source_name='orders',
    @role_name=NULL,
    @supports_net_changes=1;
```

Verify:

```SQL
SELECT name,is_cdc_enabled
FROM sys.databases
WHERE name='datapulse_demo';

SELECT name,is_tracked_by_cdc
FROM sys.tables;
```
