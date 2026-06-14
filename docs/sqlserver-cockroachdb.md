# SQL Server → CockroachDB

Use this process to prepare Datapulse for data migration from SQL Server to CockroachDB.

## 1. SQL Server Requirement

### Create Table at SQL Server

Buat database dan table di SQL Server, sebagai contoh kita akan mereplikasi 2 tables dibawah ini.
Connect using docker

```bash
docker compose -f docker/sqlserver-cockroachdb/docker-compose.yml up -d
docker compose -f docker/sqlserver-cockroachdb/docker-compose.yml down

docker exec -it sqlserver-cockroachdb-sqlserver-1 /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'SqlServer123!' -C
```

Using sqlcmd

```bash
sqlcmd -S localhost -U sa -P 'SqlServer123!' -C
```

Script membuat tables untuk sample replikasi seperti dibawah ini:

```sql
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
```

Verify:

```sql
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';
GO
```

Expected:

```
table_name
-----------------------------------------------------------------------------------
customers
orders

(2 rows affected)
```

Insert tables yang baru dibuat dengan syntax dibawah ini:

```sql
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
SQL Server -> CockroachDB
```

Karena sudah mengandung:
Primary Key, Foreign Key, VARCHAR/STRING, DECIMAL, TIMESTAMP/DATETIME, Parent-child relationship (customers → orders).

### Enable CDC pada database

```sql
USE datapulse_demo;
EXEC sys.sp_cdc_enable_db;
```

Verify:

```sql
SELECT name,is_cdc_enabled
FROM sys.databases
WHERE name='datapulse_demo';
```

Expected:

```
name                                                                                                                             is_cdc_enabled
-------------------------------------------------------------------- --------------
datapulse_demo                                                                    1

(1 rows affected)
```

Enable table CDC:

```sql
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

```sql
SELECT name,is_tracked_by_cdc
FROM sys.tables;
```

Expected:

```
name                                                           is_tracked_by_cdc
-------------------------------------------------------------- -----------------
customers                                                                      1
orders                                                                         1
systranschemas                                                                 0
change_tables                                                                  0
ddl_history                                                                    0
lsn_time_mapping                                                               0
captured_columns                                                               0
index_columns                                                                  0
dbo_customers_CT                                                               0
dbo_orders_CT                                                                  0

(10 rows affected)
```

### Enable Snapshot Isolation

Ini wajib untuk SQL Server.

```sql
ALTER DATABASE datapulse_demo
SET ALLOW_SNAPSHOT_ISOLATION ON;

--ALTER DATABASE datapulse_demo
--SET READ_COMMITTED_SNAPSHOT ON;

ALTER DATABASE datapulse_demo SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

ALTER DATABASE datapulse_demo
SET READ_COMMITTED_SNAPSHOT ON;

ALTER DATABASE datapulse_demo SET MULTI_USER;
```

Verify:

```sql
SELECT
 name,
 snapshot_isolation_state_desc,
 is_read_committed_snapshot_on
FROM sys.databases
WHERE name='datapulse_demo';
```

Expected:

```
Oname           snapshot_isolation_state_desc  is_read_committed_snapshot_on
--------------  -----------------------------  -----------------------------
datapulse_demo  ON                             1
```

### SQL Server Agent CDC Job

Verify:

```sql
SELECT name FROM msdb.dbo.sysjobs;
```

Expected:

```
name
-----------------------------------------------------------------------------------
cdc.datapulse_demo_capture
cdc.datapulse_demo_cleanup

(2 rows affected)
```

## 2. CockroachDB Requirement

Untuk CockroachDB tidak ada requirement khusus.
Connect using docker

```bash
docker exec -it sqlserver-cockroachdb-cockroachdb-1 cockroach sql --insecure
```

Using cockroach sql

```bash
cockroach sql --insecure
```

Buat database:

```sql
CREATE DATABASE datapulse_demo;
```

Buat schema:

```sql
CREATE SCHEMA dbo;
```

sesuai source database (source database SQL Server).

### 3. SQL Server → CockroachDB Flow

Export Schema

```bash
datapulse export schema \
  --source-db-type sqlserver \
  --source-db-host localhost \
  --source-db-port 1434 \
  --source-db-user sa \
  --source-db-password 'SqlServer123!' \
  --source-db-name datapulse_demo \
  --source-db-schema dbo \
  --source-ssl-mode disable \
  --export-dir /opt/datapulse/data1
```

Export Snapshot + CDC

```bash
export DATAPULSE_ENABLE_SQLSERVER_CDC=1

datapulse export data \
  --source-db-type sqlserver \
  --source-db-host localhost \
  --source-db-port 1434 \
  --source-db-user sa \
  --source-db-password 'SqlServer123!' \
  --source-db-name datapulse_demo \
  --source-db-schema dbo \
  --source-ssl-mode disable \
  --export-type snapshot-and-changes \
  --export-dir /opt/datapulse/data1
```

Expected:

```
snapshot export report
customers 10
orders    10

streaming changes to local queue
```


Import Schema

```bash
datapulse import schema \
  --target-db-type cockroachdb \
  --target-db-host localhost \
  --target-db-port 26258 \
  --target-db-user root \
  --target-db-name datapulse_demo \
  --target-db-schema dbo \
  --target-ssl-mode disable \
  --export-dir /opt/datapulse/data1
```

Import Snapshot + CDC

```bash
datapulse import data \
  --target-db-type cockroachdb \
  --target-db-host localhost \
  --target-db-port 26258 \
  --target-db-user root \
  --target-db-name datapulse_demo \
  --target-db-schema dbo \
  --target-ssl-mode disable \
  --export-dir /opt/datapulse/data1 \
  --adaptive-parallelism disabled
```

Expected:

```
snapshot import report
customers 10
orders    10

streaming changes to cockroachdb
```
