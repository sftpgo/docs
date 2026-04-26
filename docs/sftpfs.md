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
| **SFTP root directory** | Path prefix on the remote server. If set, all operations are restricted to this path and its subdirectories. Do not set this inside a symlinked directory. Defaults to `/`. |
| **Buffer size** | Buffer size in MB for read/write operations (0–16). See [Buffering](#buffering) below. |
| **Equality check mode** | How to determine if two configurations point to the same server: `0` (default) requires both endpoint and username to match; `1` requires only the endpoint to match. Used for rename operations across configurations. |
| **SOCKS proxy** | Optional SOCKS proxy address. Supported formats: `socks5://host:port`, `socks4://host:port`, `socks4a://host:port`. |
| **SOCKS username** / **SOCKS password** | Optional credentials for SOCKS proxy authentication. |

The password, private key, key passphrase, and SOCKS password are stored encrypted according to your [KMS configuration](kms.md).

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

## Connection management

SFTPGo maintains a pool of cached SFTP connections. Idle connections are reused across transfers and cleaned up after 30 seconds of inactivity. Self-connections (an SFTPFs backend pointing back to the same SFTPGo instance) are detected and rejected unless `allow_self_connections` is explicitly enabled in the [configuration](config-file.md).

## Limitations

- `chown` is not supported.
- `readlink` on cloud storage symlinks is not supported.
- Clients that require advanced filesystem-like features (e.g., `sshfs`) are not supported when buffering is enabled.
