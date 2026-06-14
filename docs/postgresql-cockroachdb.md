# PostgreSQL → CockroachDB

## 1. Create Tables at postgreSQL

Buat database dan table di PostgreSQL, sebagai contoh kita akan mereplikasi 2 tables dibawah ini.
Connect using docker

```bash
docker compose -f docker/postgresql-cockroachdb/docker-compose.yml up -d
docker compose -f docker/postgresql-cockroachdb/docker-compose.yml down

docker exec -it postgresql-cockroachdb-postgresql-1 psql -U postgres
```

Using sqlcmd

```bash
psql -U postgres
```

Script membuat tables untuk sample replikasi seperti dibawah ini:

```sql
CREATE DATABASE datapulse_demo;

\l

\c datapulse_demo;

CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(200),
    city VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    amount NUMERIC(12,2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
);
```

Verify:

```sql
\dt
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
```

Dataset ini cukup bagus untuk menguji seluruh jalur Datapulse.

```
PostgreSQL -> CockroachDB
```

Karena sudah mengandung:
Primary Key, Foreign Key, VARCHAR/STRING, DECIMAL, TIMESTAMP/DATETIME, Parent-child relationship (customers → orders).

## 2. PostgreSQL Requirement

wal_level

```sql
SHOW wal_level;
```

Expected:

```
logical
```

max_replication_slots

```sql
SHOW max_replication_slots;
```

Expected:

```
>= 1
```

max_wal_senders

```sql
SHOW max_wal_senders;
```

Expected:

```
>= 1
```

### REPLICA IDENTITY FULL

Wajib.

```sql
ALTER TABLE demo.customers
REPLICA IDENTITY FULL;

ALTER TABLE demo.orders
REPLICA IDENTITY FULL;
```

Verify:

```sql
SELECT
 n.nspname,
 c.relname,
 c.relreplident
FROM pg_class c
JOIN pg_namespace n
ON n.oid=c.relnamespace;
```

Expected:

```
f
```

## 3. CockroachDB Requirement

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

sesuai source database (source database PostgreSQL).

## 4. PostgreSQL → CockroachDB Flow
### Export Schema

```bash
datapulse export schema \
  --source-db-type postgresql \
  --source-db-host localhost \
  --source-db-port 55432 \
  --source-db-user postgres \
  --source-db-password 'postgres' \
  --source-db-name datapulse_demo \
  --source-db-schema demo \
  --source-ssl-mode disable \
  --export-dir /opt/datapulse/data2
```

### Export Snapshot + CDC

```bash
export DATAPULSE_ENABLE_POSTGRESQL_COCKROACHDB_CDC=1

datapulse export data \
  --source-db-type postgresql \
  --source-db-host localhost \
  --source-db-port 55432 \
  --source-db-user postgres \
  --source-db-password 'postgres' \
  --source-db-name datapulse_demo \
  --source-db-schema demo \
  --source-ssl-mode disable \
  --export-type snapshot-and-changes \
  --export-dir /opt/datapulse/data2
```

Expected:

```
Created replication slot
snapshot export report
streaming changes to local queue
```

### Import Schema

```bash
datapulse import schema \
  --target-db-type cockroachdb \
  --target-db-host localhost \
  --target-db-port 26259 \
  --target-db-user root \
  --target-db-name datapulse_demo \
  --target-ssl-mode disable \
  --export-dir /opt/datapulse/data2
```

Catatan:

```
JANGAN gunakan:
--target-db-schema
```

untuk source PostgreSQL.

### Import Snapshot + CDC

```bash
datapulse import data \
  --target-db-type cockroachdb \
  --target-db-host localhost \
  --target-db-port 26259 \
  --target-db-user root \
  --target-db-name datapulse_demo \
  --target-ssl-mode disable \
  --export-dir /opt/datapulse/data2 \
  --adaptive-parallelism disabled
```

Expected:

```
snapshot import report
customers 10
orders    10

Initializing streaming phase...
streaming changes to cockroachdb...
```
