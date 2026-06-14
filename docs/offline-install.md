# Datapulse Offline Install

Datapulse v1.0.1 includes an offline installer framework for air-gapped hosts.
Phase 3 added RHEL 8 and RHEL 9 RPM repository population support. Phase 4
adds Ubuntu 22.04 and Ubuntu 24.04 APT repository population support.

## Supported Targets

- RHEL 8 x86_64 and compatible distributions
- RHEL 9 x86_64 and compatible distributions
- Ubuntu 22.04 x86_64
- Ubuntu 24.04 x86_64

Use the bundle that matches the target host:

```bash
datapulse-offline-rhel8-v1.0.1-linux-amd64.tar.gz
datapulse-offline-rhel9-v1.0.1-linux-amd64.tar.gz
datapulse-offline-ubuntu22.04-v1.0.1-linux-amd64.tar.gz
datapulse-offline-ubuntu24.04-v1.0.1-linux-amd64.tar.gz
```

## Build The RHEL Dependency Repository

Run this step on an internet-connected RHEL-compatible build machine with the
required repositories already configured.

```bash
./scripts/download-rhel-deps.sh --os rhel8 --output dist/deps/rhel8
./scripts/download-rhel-deps.sh --os rhel9 --output dist/deps/rhel9
```

The script expects enabled repositories to provide:

- OpenJDK 17
- PostgreSQL client 17, including `pg_dump` and `pg_restore`
- `openssl`
- `ca-certificates`

PostgreSQL 17 packages may require the PostgreSQL official YUM repository. If
the script cannot find PostgreSQL 17 client packages, enable the PGDG repository
for the target OS and rerun the command.

If the online build host is Ubuntu or another non-RHEL platform, use the Docker
builder instead:

```bash
./scripts/run-offline-deps-builder.sh rhel8
./scripts/run-offline-deps-builder.sh rhel9
```

The Docker builders use Rocky Linux containers and write the generated RPM
repositories back to `dist/deps/rhel8` and `dist/deps/rhel9` on the host.
They also run `createrepo_c`, generate `MANIFEST.txt` and `SHA256SUMS`, and
restore ownership to the host user automatically.

## Optional Diagnostics

Optional diagnostic packages can be attempted with:

```bash
./scripts/download-rhel-deps.sh --os rhel8 --output dist/deps/rhel8 --include-optional
```

Optional packages include CockroachDB client tooling, `sqlcmd`, `msodbcsql18`,
and `mssql-tools18` when they are available from configured repositories.
Missing optional packages warn by default and fail only with
`--strict-optional`.

These packages are diagnostic-only. They are not required for Datapulse runtime.

## Build The Offline Bundle

After the repository exists under `dist/deps/<os>/rpm`, build the offline bundle:

```bash
./scripts/build-offline-bundle.sh --os rhel8 --strict-deps
./scripts/build-offline-bundle.sh --os rhel9 --strict-deps
```

For development validation, omit `--strict-deps` to build a placeholder bundle
without RPMs.

## Build The Ubuntu Dependency Repository

Run this step on an internet-connected Ubuntu-compatible build machine with the
required repositories already configured.

```bash
./scripts/download-ubuntu-deps.sh --os ubuntu22.04 --output dist/deps/ubuntu22.04
./scripts/download-ubuntu-deps.sh --os ubuntu24.04 --output dist/deps/ubuntu24.04
```

The script expects enabled repositories to provide:

- `openjdk-17-jre-headless`
- `postgresql-client-17`
- `openssl`
- `ca-certificates`

PostgreSQL 17 packages may require the PostgreSQL official APT repository. If
the script cannot find `postgresql-client-17`, enable the PGDG APT repository
for the target Ubuntu release and rerun the command.

Optional diagnostic packages can be attempted with:

```bash
./scripts/download-ubuntu-deps.sh --os ubuntu22.04 --output dist/deps/ubuntu22.04 --include-optional
```

Missing optional packages warn by default and fail only with
`--strict-optional`.

After the repository exists under `dist/deps/<os>/apt`, build the offline
bundle:

```bash
./scripts/build-offline-bundle.sh --os ubuntu22.04 --strict-deps
./scripts/build-offline-bundle.sh --os ubuntu24.04 --strict-deps
```

For development validation, omit `--strict-deps` to build a placeholder bundle
without DEBs.

If the online build host should stay clean of Ubuntu package tooling, use the
Docker builders:

```bash
./scripts/run-offline-deps-builder.sh ubuntu22.04
./scripts/run-offline-deps-builder.sh ubuntu24.04
```

The Docker builders write generated APT repositories back to
`dist/deps/ubuntu22.04` and `dist/deps/ubuntu24.04` on the host.
They also generate `Packages`, `Packages.gz`, `MANIFEST.txt`, and
`SHA256SUMS`, and restore ownership to the host user automatically.

To build dependencies and the strict offline bundle in one pass:

```bash
./scripts/run-offline-deps-builder.sh rhel9 --no-cache --build-bundle
./scripts/run-offline-deps-builder.sh ubuntu24.04 --no-cache --build-bundle
```

## Transfer To Air-Gapped Host

Copy the generated archive to the target host using your approved transfer
process:

```bash
dist/datapulse-offline-rhel8-v1.0.1-linux-amd64.tar.gz
dist/datapulse-offline-rhel9-v1.0.1-linux-amd64.tar.gz
dist/datapulse-offline-ubuntu22.04-v1.0.1-linux-amd64.tar.gz
dist/datapulse-offline-ubuntu24.04-v1.0.1-linux-amd64.tar.gz
```

## Install Without Internet

On the air-gapped RHEL host:

```bash
tar -xzf datapulse-offline-rhel8-v1.0.1-linux-amd64.tar.gz
cd datapulse-offline-rhel8-v1.0.1-linux-amd64
sudo ./install.sh --strict-deps
source /opt/datapulse/env.sh
datapulse version
datapulse doctor
```

The installer uses only `repo/rpm` from the bundle. It does not permanently
enable external repositories and does not require internet access.

On the air-gapped Ubuntu host:

```bash
tar -xzf datapulse-offline-ubuntu22.04-v1.0.1-linux-amd64.tar.gz
cd datapulse-offline-ubuntu22.04-v1.0.1-linux-amd64
sudo ./install.sh --strict-deps
source /opt/datapulse/env.sh
datapulse version
datapulse doctor
```

The installer uses only `repo/apt` from the bundle. It does not permanently
enable external repositories and does not require internet access.

## Validate

```bash
sudo /opt/datapulse/scripts/validate-install.sh --strict-deps
/opt/datapulse/scripts/doctor.sh --strict
```

## Not Included

The runtime offline bundle must not include:

- Go
- Maven
- Docker
- gcc or compiler toolchains
- build-essential equivalents
- development test fixtures
- source trees
