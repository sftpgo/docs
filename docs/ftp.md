---
description: "SFTPGo FTP/FTPS server configuration: explicit and implicit TLS, passive mode, client certificates, and TLS session reuse."
---

# FTP/FTPS

SFTPGo includes a full-featured FTP server implementation based on [RFC 959](https://datatracker.ietf.org/doc/html/rfc959){:target="_blank"} with comprehensive TLS support (FTPS).

## TLS modes

The FTP server supports four TLS modes, configured per-binding:

| Mode | Name | Description |
| ------ | ------ | ------------- |
| `0` | **Cleartext** | Plain FTP. Clients can optionally upgrade to TLS via the `AUTH TLS` command. |
| `1` | **Explicit TLS** | TLS required for both control and data connections. Clients connect in cleartext and must issue `AUTH TLS` to upgrade. |
| `2` | **Implicit TLS** | The connection is TLS-encrypted from the start (typically on port 990). |
| `3` | **Control-only TLS** | TLS required for the control connection only. Data connections remain in cleartext. |

Both a TLS certificate and private key must be configured to use explicit or implicit TLS. Certificates can be configured globally or per-binding, and support automatic renewal via [ACME](tutorials/lets-encrypt-certificate.md).

### TLS session reuse

TLS session reuse for data connections can be configured per-binding:

- `0` (default) — Not checked. Clients may or may not reuse sessions.
- `1` — Required for explicit FTPS. The client must reuse the control connection's TLS session for data connections (via session tickets; legacy session-ID reuse is not supported). Not supported with implicit TLS.
- `2` — Disabled (session tickets explicitly disabled). Not recommended for security reasons.

## Authentication

| Method | Description |
| -------- | ------------- |
| **Password** | Standard FTP password authentication. |
| **TLS certificate** | Mutual TLS authentication — the client certificate's Common Name (CN) is used as the username. Requires `client_auth_type` set to `1` (required) or `2` (optional) and at least one configured CA. |
| **Certificate + password** | Combined authentication — the client must present a valid certificate and provide the correct password. |

Per-user restrictions:

- **Denied login methods** — Restrict which authentication methods a user can use.
- **FTP security filter** — Require TLS for specific users, even when the binding allows cleartext connections.
- **Certificate pinning** — Restrict a user to specific client certificates via the TLS certificates filter.

Certificate revocation lists (CRLs) are supported and can be reloaded on demand via `SIGHUP` (Unix) or `paramchange` (Windows).

## Data connections

### Passive mode (PASV/EPSV)

In passive mode, the server opens a port for the client to connect to. Configuration options:

- **Port range** — Configurable range for passive data connections (default: 50000–50100).
- **Passive IP** — External IP address returned to clients. Options:
  - **Force passive IP** — Static IPv4 address.
  - **Passive host** — Hostname resolved on each connection (for dynamic IPs; adds DNS lookup latency).
  - **Passive IP overrides** — Return different IPs based on the client's network (useful for multi-homed servers or split-horizon setups).
- **IPv6** — Supported via the EPSV extension ([RFC 2428](https://tools.ietf.org/html/rfc2428){:target="_blank"}).

### Active mode (PORT/EPRT)

In active mode, the client opens a port and the server connects to it. Active mode is enabled by default and can be disabled via `disable_active_mode`.

By default, the server does not require port 20 for outbound connections (`active_transfers_port_non_20`), which allows running SFTPGo with fewer privileges.

### Connection security

Both active and passive modes enforce IP matching by default — the data connection must originate from (or go to) the same IP as the control connection. This prevents FTP bounce attacks. The check can be disabled per-mode if needed (e.g., on trusted internal networks).

:warning: Disabling IP matching for active connections makes the server vulnerable to bounce attacks. Only disable on trusted networks.

When running FTP behind a proxy, the PROXY protocol must be enabled for **both** control and data connections.

## Protocol extensions

### Standard extensions

| Extension | RFC | Description |
| ----------- | ----- | ------------- |
| AUTH / AUTH TLS | [2228](https://tools.ietf.org/html/rfc2228#page-6){:target="_blank"}, [4217](https://tools.ietf.org/html/rfc4217#section-4.1){:target="_blank"} | TLS session negotiation |
| PROT | [2228](https://tools.ietf.org/html/rfc2228#page-8){:target="_blank"} | Data channel encryption (PROT P = encrypted, PROT C = clear) |
| EPRT / EPSV | [2428](https://tools.ietf.org/html/rfc2428){:target="_blank"} | IPv6 support for active and passive mode |
| MDTM / MFMT | [3659](https://tools.ietf.org/html/rfc3659#page-8){:target="_blank"} | File modification time (query and set) |
| SIZE | [3659](https://tools.ietf.org/html/rfc3659#page-11){:target="_blank"} | Query file size |
| REST | [3659](https://tools.ietf.org/html/rfc3659#page-13){:target="_blank"} | Resume interrupted transfers |
| MLST / MLSD | [3659](https://tools.ietf.org/html/rfc3659#page-23){:target="_blank"} | Machine-readable file and directory listings |

### Optional extensions

These extensions are disabled by default and must be enabled in the [configuration](config-file.md):

| Extension | Config | Description |
| ----------- | -------- | ------------- |
| HASH, XCRC, MD5, XSHA* | `hash_support = 1` | File hash/checksum calculation. :warning: Reads the entire file — for remote backends this means downloading; for encrypted storage this means decrypting. |
| COMB | `combine_support = 1` | Combine multiple partial files into one. Local filesystem only — no advantage for cloud backends which support multipart uploads natively. |
| SITE (chmod, symlink) | `enable_site = true` | Enable the FTP SITE command for `chmod` and `symlink` operations. |
| AVLB | — | Query available space, respecting disk and transfer quotas. |

### Wildcard support

The `LIST` and `NLST` commands support shell-style wildcards (`*`, `?`, `[`, `]`, `^`) for filtering file names. Only single-level patterns are supported (e.g., `*.csv` works, `dir*/*.csv` does not).

## ASCII transfer mode

The FTP `TYPE` command is supported for both ASCII (`A`) and Binary (`I`) modes. By default, ASCII mode requests are honored. Set `ignore_ascii_transfer_type` to `1` to silently ignore ASCII mode requests — useful for environments with older/mainframe clients and EBCDIC files.
