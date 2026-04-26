---
description: "Use a remote FTP server as storage for SFTPGo accounts, with explicit and implicit TLS support."
---

# FTP as storage backend

A remote FTP server can be used as storage for an SFTPGo account, allowing users to transparently access files hosted on the remote server through any SFTPGo-supported protocol (SFTP, FTP, WebDAV, HTTP).

Here are the supported configuration parameters:

- `Endpoint`, FTP endpoint as `host:port`. If the port is omitted, it defaults to `21`.
- `Username`
- `Password`
- `TLS mode`, controls TLS/SSL encryption: `0` (disabled), `1` (explicit TLS via AUTH TLS), `2` (implicit TLS, typically on port 990).
- `Skip TLS verify`, if enabled, the TLS certificate presented by the remote server is not verified. Use only for testing.

The mandatory parameters are the endpoint, the username, and a password.

The password is stored as ciphertext according to your [KMS configuration](kms.md).

## TLS modes

| Mode | Description |
| ------ | ------------- |
| `0` | **Disabled** — no TLS encryption. Connection is in cleartext. |
| `1` | **Explicit TLS** — the connection starts unencrypted and is upgraded to TLS via the `AUTH TLS` command (standard on port 21). |
| `2` | **Implicit TLS** — the connection is TLS-encrypted from the start (typically on port 990). |

## Limitations

This backend has similar limitations to the [S3](s3.md) and other cloud storage backends:

- `chown`, `chmod`, `truncate`, `symlink`, and `readlink` are not supported.
- Opening a file for both reading and writing at the same time is not supported.
- Setting modification times is only supported if the remote FTP server supports the `SITE MTIME` command.
- Renaming non-empty directories is supported (via recursive rename) but may be slow for directories with many files.

## Connection behavior

SFTPGo maintains a single persistent connection per FTP backend instance. The connection is validated with a NoOp command before each operation and automatically re-established if stale. The dial timeout is 15 seconds.

To prevent connection loops, SFTPGo detects self-connections (an FTP backend pointing back to the same SFTPGo instance) and rejects them unless `allow_self_connections` is explicitly enabled in the [configuration](config-file.md).
