# Air-Gap Guide

Use this process to prepare Datapulse for isolated environments.

## Build Machine Requirements

Use an internet-connected build machine matching the target OS family:

- RHEL-compatible host for RHEL 8/9 RPM repositories
- Ubuntu-compatible host for Ubuntu 22.04/24.04 APT repositories

The build host needs package manager tooling, repository access, and checksum
tools. It should not add Go, Maven, Docker, or compiler packages to runtime
dependency repositories.

## Prepare Dependencies

RHEL:

```bash
./scripts/download-rhel-deps.sh --os rhel8 --output dist/deps/rhel8
./scripts/download-rhel-deps.sh --os rhel9 --output dist/deps/rhel9
```

Ubuntu:

```bash
./scripts/download-ubuntu-deps.sh --os ubuntu22.04 --output dist/deps/ubuntu22.04
./scripts/download-ubuntu-deps.sh --os ubuntu24.04 --output dist/deps/ubuntu24.04
```

PostgreSQL Client 17 may require the PostgreSQL official repository on the
online build machine.

## Docker-Based Dependency Builders

On an internet-connected host with Docker, use the containerized builders to
populate OS-specific repositories without installing RHEL tooling on Ubuntu or
Ubuntu tooling on RHEL:

```bash
./scripts/run-offline-deps-builder.sh rhel8
./scripts/run-offline-deps-builder.sh rhel9
./scripts/run-offline-deps-builder.sh ubuntu22.04
./scripts/run-offline-deps-builder.sh ubuntu24.04
./scripts/run-offline-deps-builder.sh all
```

The builders mount the host repository at `/workspace/datapulse` and write
packages back to `dist/deps/<os>` on the host. Optional diagnostics can be
attempted with `--include-optional`; use `--strict-optional` only when missing
diagnostic packages should fail the dependency build.

Repository finalization is automatic. RHEL builders run `createrepo_c`; Ubuntu
builders generate `Packages` and `Packages.gz`; all builders create
`MANIFEST.txt`, `SHA256SUMS`, and `README.md`, then restore ownership to the
host user. No manual metadata, checksum, manifest, or `chown` step should be
required.

To build and validate the strict offline bundle immediately after dependency
download:

```bash
./scripts/run-offline-deps-builder.sh rhel9 --no-cache --build-bundle
./scripts/run-offline-deps-builder.sh all --build-bundle
```

See `docker/offline-deps/README.md` for Docker permissions, proxy, PGDG, and
corporate CA troubleshooting.

## Build Offline Bundles

```bash
./scripts/build-minimal-release.sh
./scripts/build-offline-bundle.sh --os rhel8 --strict-deps
./scripts/build-offline-bundle.sh --os rhel9 --strict-deps
./scripts/build-offline-bundle.sh --os ubuntu22.04 --strict-deps
./scripts/build-offline-bundle.sh --os ubuntu24.04 --strict-deps
```

Use non-strict mode only for framework validation without packaged
dependencies.

## Transfer And Verify

Transfer the offline tarball and checksum files through the approved air-gap
process. On the isolated host:

```bash
sha256sum -c SHA256SUMS
tar -xzf datapulse-offline-<os>-v1.0.1-linux-amd64.tar.gz
```

## Install, Upgrade, Uninstall

```bash
sudo ./install.sh --strict-deps
sudo /opt/datapulse/scripts/validate-install.sh --strict-deps
sudo ./upgrade.sh
sudo ./uninstall.sh
sudo ./uninstall.sh --purge
```

## Troubleshooting

- Empty repository in strict mode: rebuild dependencies on the online host.
- Missing PostgreSQL 17 client: enable the PGDG repository and rerun the
  dependency download script.
- Checksum failure: retransmit the artifact from the trusted build location.
- Install validation warnings: run `datapulse doctor --json` and capture logs.
