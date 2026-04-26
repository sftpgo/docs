---
description: "Filter SFTPGo connections by country using MaxMind or IPLocation.io GeoIP databases. Allow or deny access based on geographic location."
---

# GeoIP Filtering

The GeoIP filter plugin allows you to accept or deny connections based on the geographic location of the client's IP address.

The plugin uses [MMDB](https://maxmind.github.io/MaxMind-DB/){:target="_blank"} (MaxMind DB) format databases to resolve IP addresses to country codes.

## Supported databases

- [MaxMind GeoLite2](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data){:target="_blank"} (free, requires registration)
- [MaxMind GeoIP2](https://www.maxmind.com/en/geoip-databases){:target="_blank"} (commercial, higher accuracy)
- [IPLocation.io](https://www.iplocation.io/){:target="_blank"} MMDB databases

## Installation

Install the `sftpgo-plugins` package as described in [Audit Logs - Installation](audit-logs.md#installation). The plugin binary is `sftpgo-plugin-geoipfilter`.

## Configuration

:warning: Any configuration change described below requires a service restart to take effect (e.g. `systemctl restart sftpgo`).

### Step 1: Obtain an MMDB database

Download a GeoLite2 Country database from [MaxMind](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data){:target="_blank"} or an MMDB database from [IPLocation.io](https://www.iplocation.io/){:target="_blank"} and place it on your server, for example at `/var/lib/sftpgo/GeoLite2-Country.mmdb`.

### Step 2: Configure the plugin

Specify the database path and either a list of allowed countries or denied countries. Country codes use the [ISO 3166-1 alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2){:target="_blank"} format (two-letter codes).

**Allow only specific countries:**

```shell
SFTPGO_PLUGIN_GEOIPFILTER_DB_FILE="/var/lib/sftpgo/GeoLite2-Country.mmdb"
SFTPGO_PLUGIN_GEOIPFILTER_ALLOWED_COUNTRIES="IT,US,DE"

SFTPGO_PLUGINS__0__TYPE=ipfilter
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-geoipfilter"
SFTPGO_PLUGINS__0__ARGS="serve"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

**Deny specific countries (allow all others):**

```shell
SFTPGO_PLUGIN_GEOIPFILTER_DB_FILE="/var/lib/sftpgo/GeoLite2-Country.mmdb"
SFTPGO_PLUGIN_GEOIPFILTER_DENIED_COUNTRIES="CN,RU"

SFTPGO_PLUGINS__0__TYPE=ipfilter
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-geoipfilter"
SFTPGO_PLUGINS__0__ARGS="serve"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

You must specify either `--allowed-countries` or `--denied-countries`, not both.

:information_source: The examples above use plugin index `0`. If you have other plugins already configured, adjust the index accordingly. See [Plugin indexing](../plugins.md#plugin-indexing) for details.

### Using IPLocation.io databases

If you use an IPLocation.io MMDB database instead of MaxMind, set the database type:

```shell
SFTPGO_PLUGIN_GEOIPFILTER_DB_TYPE=1
```

The default (`0`) is for MaxMind-compatible databases.

## Behavior

- **Private IP addresses** (RFC 1918, loopback, link-local) are always allowed, regardless of country filters.
- If the **country lookup fails** (IP not found in database, database read error), the connection is **allowed** by default.
- The database can be **reloaded without restart** by sending a reload command to SFTPGo. This is useful when updating the MMDB file.

## Configuration reference

| Environment variable | Flag | Description |
| -------------------- | ---- | ----------- |
| `SFTPGO_PLUGIN_GEOIPFILTER_DB_FILE` | `--db-file` | Path to the MMDB database file (required) |
| `SFTPGO_PLUGIN_GEOIPFILTER_DB_TYPE` | `--db-type` | Database type: `0` = MaxMind (default), `1` = IPLocation.io |
| `SFTPGO_PLUGIN_GEOIPFILTER_ALLOWED_COUNTRIES` | `--allowed-countries` | Comma-separated ISO 3166-1 alpha-2 country codes to allow |
| `SFTPGO_PLUGIN_GEOIPFILTER_DENIED_COUNTRIES` | `--denied-countries` | Comma-separated ISO 3166-1 alpha-2 country codes to deny |
