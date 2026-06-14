# PostgreSQL → CockroachDB

## 1. PostgreSQL Requirement

wal_level

```SQL
SHOW wal_level;
```

Expected:

```
logical
```

max_replication_slots

```SQL
SHOW max_replication_slots;
```

Expected:

```
>= 1
```

max_wal_senders

```SQL
SHOW max_wal_senders;
```

Expected:

```
>= 1
```

### REPLICA IDENTITY FULL

Wajib.

```SQL
ALTER TABLE demo.customers
REPLICA IDENTITY FULL;

ALTER TABLE demo.orders
REPLICA IDENTITY FULL;
```

Verify:

```SQL
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

## 2. PostgreSQL → CockroachDB Flow
### Export Schema

```BASH
datapulse export schema \
  --source-db-type postgresql \
  --source-db-host localhost \
  --source-db-port 55432 \
  --source-db-user postgres \
  --source-db-password 'postgres' \
  --source-db-name demo \
  --source-db-schema demo \
  --source-ssl-mode disable \
  --export-dir /opt/datapulse/data2
```

### Export Snapshot + CDC

```BASH
export DATAPULSE_ENABLE_POSTGRESQL_COCKROACHDB_CDC=1

datapulse export data \
  --source-db-type postgresql \
  --source-db-host localhost \
  --source-db-port 55432 \
  --source-db-user postgres \
  --source-db-password 'postgres' \
  --source-db-name demo \
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

```BASH
datapulse import schema \
  --target-db-type cockroachdb \
  --target-db-host localhost \
  --target-db-port 26259 \
  --target-db-user root \
  --target-db-name demo \
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

```BASH
datapulse import data \
  --target-db-type cockroachdb \
  --target-db-host localhost \
  --target-db-port 26259 \
  --target-db-user root \
  --target-db-name demo \
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
