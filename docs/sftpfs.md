---
description: "Use a remote SFTP server as storage for SFTPGo accounts, with optional buffering for improved performance and SOCKS proxy support."
---

# SFTP as storage backend

A remote SFTP server can be used as storage for an SFTPGo account, allowing users to transparently access files hosted on the remote server through any SFTPGo-supported protocol.

## Configuration

| Parameter | Description |
| ----------- | ------------- |
| **Endpoint** | SSH endpoint as `host:port`. If the port is omitted, it defaults to `22`. Required. |
| **Username** | Username on the remote SFTP server. Required. |
| **Password** | SSH password. At least one of password or private key must be provided. If both are set, the private key is tried first. |
| **Private key** | SSH private key in PEM format (see example below). At least one of password or private key must be provided. |
| **Key passphrase** | Passphrase for encrypted private keys. Only needed if the private key is passphrase-protected. |
| **Fingerprints** | SHA256 fingerprints of the remote server's host key. Optional but highly recommended: if provided, the connection is rejected unless one of the fingerprints matches the server's host key. |
| **SFTP root directory** | Path prefix on the remote server. If set, the account is scoped to this path and its subdirectories — see [SFTP root directory and path confinement](#sftp-root-directory-and-path-confinement) for what this boundary guarantees. Do not set it to a symbolic link or inside a symlinked directory. Defaults to `/`. |
| **Buffer size** | Buffer size in MB for read/write operations (0–16). See [Buffering](#buffering) below. |
| **Disable concurrent writes** | Forward each write request to the remote server sequentially. Enable for servers that do not support SFTP pipelining. |
| **Equality check mode** | How to determine if two configurations point to the same server: `0` (default) requires both endpoint and username to match; `1` requires only the endpoint to match. Used for rename operations across configurations. |
| **SOCKS proxy** | Optional SOCKS proxy address. Supported formats: `socks5://host:port`, `socks4://host:port`, `socks4a://host:port`. |
| **SOCKS username** / **SOCKS password** | Optional credentials for SOCKS proxy authentication. |

The password, private key, key passphrase, and SOCKS password are stored encrypted according to your [KMS configuration](kms.md).

## SFTP root directory and path confinement

When an **SFTP root directory** (prefix) other than `/` is set, SFTPGo scopes the account to that path and its subdirectories on the remote server. In that case — and only then — SFTPGo resolves symbolic links in every component of each requested path against the remote server as a best-effort check, so a link that persistently points outside the prefix is rejected rather than followed. This is the same model described in [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions); the per-directory permission interaction documented there applies here too.

Understand what this boundary does and does not guarantee:

- The prefix is layered on a remote account that SFTPGo reaches with the configured credentials, which can access the whole account. The prefix scopes the SFTPGo *user* to a subtree; it is not a separate account.
- Enforcement depends on SFTPGo resolving paths on the remote, so it assumes the remote server reports file types and link targets faithfully. On a remote server you do not control, treat the prefix as a scoping convenience, not a hard isolation boundary.
- For strict isolation between tenants, prefer a separate remote account per tenant over different prefixes on a shared account.
- The path is resolved and then operated on in separate steps, and the SFTP protocol has no atomic rooted operation, so a residual time-of-check/time-of-use window remains that cannot be fully closed on a remote backend: a client holding `create_symlinks` could, by winning a race, have a link followed past the prefix. Any such access stays within the remote account the configured credentials already reach. To remove this window on a remote you control, start from a tree with no symbolic links and keep symbolic-link creation disabled (the default `symlink_mode`); for isolation between tenants, prefer a separate account over a prefix (see above).

With the default prefix `/`, the whole remote account is the root: SFTPGo performs no per-component symlink resolution and the remote server's own boundary is what applies.

### Private key format

The private key should be PEM encoded:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACA8LWc4SahqKkAr4L3rS19w1Vt8/IAf4th2FZmf+PJ/vwAAAJBvnZIJb52S
CQAAAAtzc2gtZWQyNTUxOQAAACA8LWc4SahqKkAr4L3rS19w1Vt8/IAf4th2FZmf+PJ/vw
AAAEBE6F5Az4wzNfNYLRdG8blDwvPBYFXE8BYDi4gzIhnd9zwtZzhJqGoqQCvgvetLX3DV
W3z8gB/i2HYVmZ/48n+/AAAACW5pY29sYUBwMQECAwQ=
-----END OPENSSH PRIVATE KEY-----
```

## Buffering

By default (`Buffer size = 0`), SFTPGo communicates with the remote SFTP server using direct, unbuffered I/O.

When buffering is enabled (buffer size 1–16 MB), reads and writes are split into multiple concurrent requests. This improves transfer performance over high-latency networks by overlapping round-trip times.

However, enabling buffering has trade-offs:

- Upload resume is not supported.
- Atomic uploads are not supported.
- A file cannot be opened for both reading and writing at the same time.

Some SFTP servers (e.g., AWS Transfer) do not support opening files in read/write mode simultaneously. Enabling buffering provides a workaround.

## Concurrent reads

By default, SFTPGo uses concurrent reads when downloading files from the remote SFTP server. Some SFTP servers automatically delete files after a download — if you use such a server, disable concurrent reads to ensure compatibility.

## Concurrent writes

Writes to the remote server can be sent as concurrent requests (SFTP pipelining) — when buffering is enabled, or when a client uploads over SFTP using multiple parallel write requests. This keeps throughput high over high-latency links, but some servers accept only one write request at a time. With such a server, enable **Disable concurrent writes** to guarantee that write requests are sent to the remote server one at a time.

## Connection management

SFTPGo maintains a pool of cached SFTP connections. Idle connections are reused across transfers and cleaned up after 30 seconds of inactivity. Self-connections (an SFTPFs backend pointing back to the same SFTPGo instance) are detected and rejected unless `allow_self_connections` is explicitly enabled in the [configuration](config-file.md).

## Limitations

- `chown` is not supported.
- `readlink` on cloud storage symlinks is not supported.
- Clients that require advanced filesystem-like features (e.g., `sshfs`) are not supported when buffering is enabled.
