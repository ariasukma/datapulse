# Datapulse 

This directory is the production documentation subset for Datapulse v1.0.1.
It excludes internal planning notes, phase histories, release signoff drafts,
test fixtures. For data replication for database.

Datapulse is a universal database replication and migration platform.

## Project Identity

Product Name: Datapulse
Migration Engine: Datapulse migration engine
CLI: datapulse

Future goals:
- Oracle
- MySQL
- MariaDB
- SQL Server
- PostgreSQL
- YugabyteDB
- CockroachDB
- MongoDB

Current status:
- Snapshot export/import working
- Docker integration tests passing
- SQL Server CDC snapshot-and-changes guarded behind DATAPULSE_ENABLE_SQLSERVER_CDC=1
