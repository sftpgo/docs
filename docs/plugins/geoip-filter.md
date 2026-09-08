---
description: "Filter SFTPGo connections by country using MaxMind, IPLocate or IPinfo GeoIP databases. Allow or deny access based on geographic location."
---

# GeoIP Filtering

The GeoIP filter plugin allows you to accept or deny connections based on the geographic location of the client's IP address.

The plugin uses [MMDB](https://maxmind.github.io/MaxMind-DB/){:target="_blank"} (MaxMind DB) format databases to resolve IP addresses to country codes.

## Supported databases

- [MaxMind GeoLite2](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data){:target="_blank"} (free, requires registration)
- [MaxMind GeoIP2](https://www.maxmind.com/en/geoip-databases){:target="_blank"} (commercial, higher accuracy)
- [IPLocate](https://www.iplocate.io/){:target="_blank"} ip-to-country MMDB databases
- [IPinfo Lite](https://ipinfo.io/products/free-ip-database){:target="_blank"} MMDB databases (free, requires registration, updated daily)

## Installation

Install the `sftpgo-plugins` package as described in [Audit Logs - Installation](audit-logs.md#installation). The plugin binary is `sftpgo-plugin-geoipfilter`.

## Configuration

:warning: Any configuration change described below requires a service restart to take effect (e.g. `systemctl restart sftpgo`).

### Step 1: Obtain an MMDB database

Download a GeoLite2 Country database from [MaxMind](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data){:target="_blank"}, an ip-to-country MMDB database from [IPLocate](https://www.iplocate.io/){:target="_blank"} or an IPinfo Lite MMDB database from [IPinfo](https://ipinfo.io/products/free-ip-database){:target="_blank"} and place it on your server, for example at `/var/lib/sftpgo/GeoLite2-Country.mmdb`.

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

You must specify at least one between `--allowed-countries` and `--denied-countries`. You can also set both: denied countries are checked first and take precedence.

:information_source: The examples above use plugin index `0`. If you have other plugins already configured, adjust the index accordingly. See [Plugin indexing](../plugins.md#plugin-indexing) for details.

### Using IPLocate or IPinfo databases

If you use an IPLocate or IPinfo Lite MMDB database instead of MaxMind, set the database type:

```shell
SFTPGO_PLUGIN_GEOIPFILTER_DB_TYPE=1
```

The default (`0`) is for MaxMind-compatible databases, while `1` is for databases that store the country code in a top-level `country_code` field, such as IPLocate and IPinfo. The plugin will not start if `db-type` is set to an unsupported value.

### Fail-closed mode

By default the plugin fails open: if the country of an IP address cannot be determined, the connection is allowed. Starting from plugin version 1.2.0 you can change this behavior with two separate options:

```shell
SFTPGO_PLUGIN_GEOIPFILTER_DENY_ON_LOOKUP_ERROR=1
SFTPGO_PLUGIN_GEOIPFILTER_DENY_UNKNOWN_COUNTRY=1
```

- `DENY_ON_LOOKUP_ERROR` denies the connection if the IP address cannot be parsed or the database lookup fails, for example because the database is corrupt or an IPv6 address is looked up in an IPv4-only database.
- `DENY_UNKNOWN_COUNTRY` denies the connection if the lookup succeeds but no country is found, for example because the IP is not in the database or the matching record has no country code. This is recommended when using `ALLOWED_COUNTRIES`, so that unclassifiable public IPs cannot bypass the allow list.

Non-public IP addresses are always allowed, so enabling these options does not affect local or private network access.

## Behavior

- **Non-public IP addresses** (RFC 1918 private ranges, loopback, link-local, shared address space/CGNAT) are always allowed, regardless of country filters.
- If the **country lookup fails** (IP not found in database, database read error), the connection is **allowed** by default. See [Fail-closed mode](#fail-closed-mode) to change this.
- The database can be **reloaded without restart** by sending a reload command to SFTPGo. This is useful when updating the MMDB file. If the reload fails, the previously loaded database is preserved and remains in use.

## Configuration reference

| Environment variable | Flag | Description |
| -------------------- | ---- | ----------- |
| `SFTPGO_PLUGIN_GEOIPFILTER_DB_FILE` | `--db-file` | Path to the MMDB database file (required) |
| `SFTPGO_PLUGIN_GEOIPFILTER_DB_TYPE` | `--db-type` | Database type: `0` = MaxMind (default), `1` = IPLocate/IPinfo |
| `SFTPGO_PLUGIN_GEOIPFILTER_ALLOWED_COUNTRIES` | `--allowed-countries` | Comma-separated ISO 3166-1 alpha-2 country codes to allow |
| `SFTPGO_PLUGIN_GEOIPFILTER_DENIED_COUNTRIES` | `--denied-countries` | Comma-separated ISO 3166-1 alpha-2 country codes to deny |
| `SFTPGO_PLUGIN_GEOIPFILTER_DENY_ON_LOOKUP_ERROR` | `--deny-on-lookup-error` | Deny the IP if it cannot be parsed or the lookup fails. Default: allowed |
| `SFTPGO_PLUGIN_GEOIPFILTER_DENY_UNKNOWN_COUNTRY` | `--deny-unknown-country` | Deny the IP if no country is found in the database. Default: allowed |
