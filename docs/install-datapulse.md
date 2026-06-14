# Install Datapulse

## Install From The Minimal Bundle

Extract the release archive and run the bundled installer:

```bash
tar -xzf datapulse-v1.0.1-linux-amd64.tar.gz
cd datapulse
sudo ./install.sh
datapulse version
```

Validate the installed layout:

```bash
./validate-install.sh
```

Upgrade from a newly extracted minimal bundle:

```bash
sudo ./upgrade.sh
```

Uninstall:

```bash
sudo ./uninstall.sh
```

The minimal v1.0.1 bundle is directly installable, but it does not include
offline OS dependency repositories or download RPM or DEB packages. Use the
OS-specific offline bundles when dependency installation must also work without
internet access.

## Recommended Directories

- Config: `/etc/datapulse`
- Runtime state: `/var/lib/datapulse`
- Logs: migration export directory `logs/` subdirectories

## Help Commands

Supported help forms:

```bash
datapulse -h
datapulse --help
datapulse -help
```
