# Datapulse Quickstart

Datapulse is invoked with the `datapulse` command.

## Validate The CLI

```bash
datapulse -h
datapulse --help
datapulse version
```

`datapulse -help` is accepted as a compatibility help alias.

## Snapshot Workflow

Use a dedicated export directory for each migration.

```bash
mkdir -p /var/lib/datapulse/migrations/example

datapulse export schema --export-dir /var/lib/datapulse/migrations/example
datapulse import schema --export-dir /var/lib/datapulse/migrations/example
datapulse export data --export-dir /var/lib/datapulse/migrations/example
datapulse import data --export-dir /var/lib/datapulse/migrations/example
```

Provide source and target connection settings with CLI flags, environment
variables, or a Datapulse config file.

## CDC Workflow

CDC is opt-in and requires the supported source/target path plus the required
runtime flags documented for that path.

```bash
DATAPULSE_ENABLE_SQLSERVER_CDC=1 datapulse export data --export-dir /var/lib/datapulse/migrations/example
datapulse import data --export-dir /var/lib/datapulse/migrations/example
```

## Logs

New runs write Datapulse-branded logs under the migration export directory:

- `logs/datapulse-export-schema.log`
- `logs/datapulse-import-schema.log`
- `logs/datapulse-export-data.log`
- `logs/datapulse-import-data.log`
- `logs/datapulse-assess-migration.log`
