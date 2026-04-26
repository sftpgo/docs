---
description: "Practical configuration examples for SFTPGo: SFTP on a custom port, FTP with TLS, WebDAV, cloud storage, proxy protocol, and more."
---

# Configuration Examples

This page provides examples for common SFTPGo configuration tasks. For the full configuration reference, see [Configuration file](config-file.md) and [Environment variables](env-vars.md).

SFTPGo can be configured via a JSON, TOML, or YAML configuration file, or via environment variables. **We recommend using environment variables** — the configuration file format may change between versions, while environment variables remain stable.

On Linux with official packages, set environment variables in `/etc/sftpgo/sftpgo.env` or in individual files under `/etc/sftpgo/env.d/`. On Windows, the configuration directory is typically `C:\ProgramData\SFTPGo Enterprise`.

The examples below show both approaches. After any configuration change, restart SFTPGo to apply it.

## Database providers

The default configuration uses SQLite (or bolt on some architectures). For production deployments, especially with multiple SFTPGo instances, switch to PostgreSQL, MySQL, or CockroachDB.

### PostgreSQL

Create the database and user:

```shell
sudo -i -u postgres psql
CREATE DATABASE "sftpgo" WITH ENCODING='UTF8' CONNECTION LIMIT=-1;
CREATE USER "sftpgo" WITH ENCRYPTED PASSWORD 'your password here';
GRANT ALL PRIVILEGES ON DATABASE "sftpgo" TO "sftpgo";
\q
```

Configure SFTPGo using environment variables (create `/etc/sftpgo/env.d/postgresql.env`):

```shell
SFTPGO_DATA_PROVIDER__DRIVER=postgresql
SFTPGO_DATA_PROVIDER__NAME=sftpgo
SFTPGO_DATA_PROVIDER__HOST=127.0.0.1
SFTPGO_DATA_PROVIDER__PORT=5432
SFTPGO_DATA_PROVIDER__USERNAME=sftpgo
SFTPGO_DATA_PROVIDER__PASSWORD=your password here
```

Or equivalently in the configuration file (`data_provider` section):

```json
"data_provider": {
    "driver": "postgresql",
    "name": "sftpgo",
    "host": "127.0.0.1",
    "port": 5432,
    "username": "sftpgo",
    "password": "your password here"
}
```

If PostgreSQL runs on the same host as SFTPGo and both are managed by systemd (typical Linux VM / bare-metal deployment), ensure SFTPGo starts after the database:

```shell
sudo systemctl edit sftpgo.service
```

```shell
[Unit]
After=postgresql.service
```

This step does not apply when PostgreSQL is remote (different host), or when SFTPGo runs in Docker or Kubernetes — in those cases startup ordering is handled by the container runtime / orchestrator (or by SFTPGo's own retry logic on initial connection).

### MySQL / MariaDB

Create the database and user:

```shell
mysql -u root
CREATE DATABASE sftpgo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON sftpgo.* TO sftpgo@localhost IDENTIFIED BY 'your password here';
\q
```

Configure SFTPGo (create `/etc/sftpgo/env.d/mysql.env`):

```shell
SFTPGO_DATA_PROVIDER__DRIVER=mysql
SFTPGO_DATA_PROVIDER__NAME=sftpgo
SFTPGO_DATA_PROVIDER__HOST=127.0.0.1
SFTPGO_DATA_PROVIDER__PORT=3306
SFTPGO_DATA_PROVIDER__USERNAME=sftpgo
SFTPGO_DATA_PROVIDER__PASSWORD=your password here
```

If MySQL / MariaDB runs on the same host as SFTPGo and both are managed by systemd (typical Linux VM / bare-metal deployment), ensure SFTPGo starts after the database:

```shell
sudo systemctl edit sftpgo.service
```

```shell
[Unit]
After=mariadb.service
```

This step does not apply when the database is remote (different host), or when SFTPGo runs in Docker or Kubernetes — in those cases startup ordering is handled by the container runtime / orchestrator (or by SFTPGo's own retry logic on initial connection).

### CockroachDB

Create the database:

```shell
cockroach sql --certs-dir=/etc/cockroach/certs -e 'CREATE DATABASE "sftpgo"'
```

Configure SFTPGo (create `/etc/sftpgo/env.d/cockroachdb.env`):

```shell
SFTPGO_DATA_PROVIDER__DRIVER=cockroachdb
SFTPGO_DATA_PROVIDER__NAME=sftpgo
SFTPGO_DATA_PROVIDER__HOST=localhost
SFTPGO_DATA_PROVIDER__PORT=26257
SFTPGO_DATA_PROVIDER__USERNAME=root
SFTPGO_DATA_PROVIDER__SSLMODE=3
SFTPGO_DATA_PROVIDER__ROOT_CERT="/etc/cockroach/certs/ca.crt"
SFTPGO_DATA_PROVIDER__CLIENT_CERT="/etc/cockroach/certs/client.root.crt"
SFTPGO_DATA_PROVIDER__CLIENT_KEY="/etc/cockroach/certs/client.root.key"
```

:information_source: CockroachDB does not implement `pg_advisory_lock`, so SFTPGo cannot serialize schema migrations against it. When running multi-node, either start instances sequentially or set `update_mode: 1` on every instance and run `initprovider` from a single operator job before the rollout.

### Verifying the connection

After configuring any database provider, verify the connection:

```shell
sudo su - sftpgo -s /bin/bash -c 'sftpgo initprovider -c /etc/sftpgo'
```

SFTPGo also attempts to initialize and update the database automatically on startup.

## Enable FTP service

The FTP service is disabled by default. To enable it, set the port:

```shell
SFTPGO_FTPD__BINDINGS__0__PORT=2121
```

The FTP service is now available on port `2121`.

For passive FTP, the default port range is `50000-50100` — these ports must be reachable by clients. If SFTPGo is behind NAT, set `force_passive_ip` to your external IP:

```shell
SFTPGO_FTPD__BINDINGS__0__FORCE_PASSIVE_IP=203.0.113.10
SFTPGO_FTPD__PASSIVE_PORT_RANGE__START=50000
SFTPGO_FTPD__PASSIVE_PORT_RANGE__END=50100
```

It is strongly recommended to enable TLS. You can configure a certificate file or use the built-in [Let's Encrypt](tutorials/lets-encrypt-certificate.md) integration. For details, see [FTP configuration](ftp.md).

## Enable WebDAV service

The WebDAV service is disabled by default. To enable it, set the port:

```shell
SFTPGO_WEBDAVD__BINDINGS__0__PORT=10080
```

The WebDAV service is now available on port `10080`. It is strongly recommended to enable HTTPS — configure a certificate or use [Let's Encrypt](tutorials/lets-encrypt-certificate.md). For details, see [WebDAV configuration](webdav.md).
