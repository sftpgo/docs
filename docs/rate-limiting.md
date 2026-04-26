---
description: "Configure per-protocol and per-IP rate limiting in SFTPGo to protect against brute-force attacks and request flooding."
---

# Rate limiting

Rate limiting controls the number of requests reaching SFTPGo services. It helps protect against brute-force attacks, credential stuffing, and accidental overload.

SFTPGo implements a [token bucket](https://en.wikipedia.org/wiki/Token_bucket){:target="_blank"} algorithm. The bucket starts full and is refilled at the configured rate. The `burst` parameter defines the bucket size (the maximum number of requests that can be served instantly). The rate is calculated by dividing `average` by `period` — for example, `average=100` and `period=1000` (milliseconds) results in 100 requests per second.

Requests that exceed the configured limit are delayed. If the required delay exceeds the maximum allowed (internally computed, capped at 10 seconds), the request is rejected.

## Protocols

Rate limiters are configured per-protocol. The supported protocols are:

- **SSH** — includes SFTP and SSH commands
- **FTP** — includes FTP, FTPES, FTPS
- **DAV** — WebDAV
- **HTTP** — REST API and web interfaces

## Limiter types

You can define two types of rate limiters:

- **Global** (`type: 1`) — applies an aggregate limit across all clients for the configured protocols. Useful for protecting overall server capacity.
- **Per-host** (`type: 2`) — maintains a separate rate limiter for each client IP address. Can be connected to the built-in [defender](defender.md) to automatically block hosts that repeatedly exceed the limit.

For per-host limiters, SFTPGo keeps a rate limiter in memory for each connecting host. Use `entries_soft_limit` and `entries_hard_limit` to control memory usage: when the number of tracked hosts exceeds the soft limit, the least recently used entries are removed; the hard limit is the absolute maximum.

## Exemptions

You can exempt IP addresses from rate limiting in two ways:

- Add them to the **Rate limiters safe list** via the WebAdmin UI or REST API.
- Add them to the **Trusted list**, which also exempts them from other restrictions.

In multi-node setups, list entry propagation between nodes may take some minutes.

## Configuration

Rate limiting is disabled by default (`average: 0`). You can define multiple rate limiters — each request is checked against all matching limiters.

The following example defines two rate limiters:

1. A **global** limiter allowing 100 requests/second across all protocols.
2. A **per-host** limiter allowing 10 requests/second per IP for SSH and FTP, with defender integration enabled.

### Using environment variables (recommended)

Create the file `/etc/sftpgo/env.d/rate-limiting.env`:

```shell
# Global rate limiter: 100 req/s for all protocols
SFTPGO_COMMON__RATE_LIMITERS__0__AVERAGE=100
SFTPGO_COMMON__RATE_LIMITERS__0__PERIOD=1000
SFTPGO_COMMON__RATE_LIMITERS__0__BURST=1
SFTPGO_COMMON__RATE_LIMITERS__0__TYPE=1
SFTPGO_COMMON__RATE_LIMITERS__0__PROTOCOLS=SSH,FTP,DAV,HTTP
SFTPGO_COMMON__RATE_LIMITERS__0__GENERATE_DEFENDER_EVENTS=0
SFTPGO_COMMON__RATE_LIMITERS__0__ENTRIES_SOFT_LIMIT=100
SFTPGO_COMMON__RATE_LIMITERS__0__ENTRIES_HARD_LIMIT=150

# Per-host rate limiter: 10 req/s per IP for SSH and FTP
SFTPGO_COMMON__RATE_LIMITERS__1__AVERAGE=10
SFTPGO_COMMON__RATE_LIMITERS__1__PERIOD=1000
SFTPGO_COMMON__RATE_LIMITERS__1__BURST=1
SFTPGO_COMMON__RATE_LIMITERS__1__TYPE=2
SFTPGO_COMMON__RATE_LIMITERS__1__PROTOCOLS=SSH,FTP
SFTPGO_COMMON__RATE_LIMITERS__1__GENERATE_DEFENDER_EVENTS=1
SFTPGO_COMMON__RATE_LIMITERS__1__ENTRIES_SOFT_LIMIT=100
SFTPGO_COMMON__RATE_LIMITERS__1__ENTRIES_HARD_LIMIT=150
```

### Using the configuration file

```json
{
  "common": {
    "rate_limiters": [
      {
        "average": 100,
        "period": 1000,
        "burst": 1,
        "type": 1,
        "protocols": ["SSH", "FTP", "DAV", "HTTP"],
        "generate_defender_events": false,
        "entries_soft_limit": 100,
        "entries_hard_limit": 150
      },
      {
        "average": 10,
        "period": 1000,
        "burst": 1,
        "type": 2,
        "protocols": ["SSH", "FTP"],
        "generate_defender_events": true,
        "entries_soft_limit": 100,
        "entries_hard_limit": 150
      }
    ]
  }
}
```

With this configuration, SSH and FTP clients are checked first against the global limiter (100 req/s total) and then against the per-host limiter (10 req/s per IP). WebDAV and HTTP clients are only checked against the global limiter.

## Configuration reference

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `average` | integer | `0` | Maximum number of requests allowed in the given period. `0` disables the rate limiter. |
| `period` | integer | `1000` | Time window in milliseconds. Must be at least `100`. |
| `burst` | integer | `1` | Maximum number of requests that can be served instantly (bucket size). Must be at least `1`. |
| `type` | integer | `2` | `1` = global (aggregate across all clients), `2` = per-host (per client IP). |
| `protocols` | list of strings | all | Protocols this limiter applies to: `SSH`, `FTP`, `DAV`, `HTTP`. |
| `generate_defender_events` | boolean | `false` | Generate defender events when the limit is exceeded. Only meaningful for per-host limiters. |
| `entries_soft_limit` | integer | `100` | Per-host only. Number of tracked hosts after which least recently used entries are evicted. |
| `entries_hard_limit` | integer | `150` | Per-host only. Absolute maximum number of tracked hosts. Must be greater than `entries_soft_limit`. |

Full configuration details are available in the [configuration file reference](config-file.md).
