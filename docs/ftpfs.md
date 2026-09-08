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
- `Remote directory`, optional server-side path used as the starting directory for all operations. See [Remote directory](#remote-directory) below.

The mandatory parameters are the endpoint, the username, and a password.

The password is stored as ciphertext according to your [KMS configuration](kms.md).

## TLS modes

| Mode | Description |
| ------ | ------------- |
| `0` | **Disabled** — no TLS encryption. Connection is in cleartext. |
| `1` | **Explicit TLS** — the connection starts unencrypted and is upgraded to TLS via the `AUTH TLS` command (standard on port 21). |
| `2` | **Implicit TLS** — the connection is TLS-encrypted from the start (typically on port 990). |

## Remote directory

By default, operations are relative to the working directory delivered by the FTP server after login. Set a `Remote directory` to scope the backend to a sub-path of that tree, so users land directly in the right place. For example, with a remote directory of `/exports/reports`, a user listing `/` sees the contents of that path on the remote server.

:warning: The remote directory is **not a security boundary**. SFTPGo cannot enforce it the way the [SFTP backend](sftpfs.md) enforces its prefix: a server-side symlink under the remote directory may resolve to a location outside it, and FTP gives SFTPGo no reliable way to detect this. Treat the remote directory as a convenience starting path, not a confinement.

Because of this, the feature is opt-in at the deployment level: the `ftp` backend must be listed in the [`allow_remote_directory`](config-file.md) setting of the `common` configuration section. The remote directory is honored only while the backend is enabled — a connection using a stored configuration whose backend is not in the allow list is rejected, with the reason logged. The value is preserved on save and on backup restore, so disabling the backend does not break data import; in the WebAdmin the field is shown when the backend is enabled, or with a warning when a value is set while the backend is disabled, so it can be cleared.

When the backend is defined at the group level, the remote directory supports the same [placeholders](groups.md#placeholders) as the cloud key prefix and the SFTP prefix. The placeholders are resolved per user when the group settings are applied, so a single group configuration can scope each member to their own sub-path.

## Limitations

This backend has similar limitations to the [S3](s3.md) and other cloud storage backends:

- `chown`, `chmod`, `truncate`, `symlink`, and `readlink` are not supported.
- Opening a file for both reading and writing at the same time is not supported.
- Setting modification times is supported when the remote FTP server advertises the `MFMT` feature.
- Renaming non-empty directories is supported (via recursive rename) but may be slow for directories with many files.

## Connection behavior

An FTP backend reuses one connection to the remote server across operations, checked with a NoOp command before each use and re-established when the server has dropped it. FTP carries one operation at a time and a transfer holds its connection for its whole duration, so operations that overlap take a connection each.

Size the connection limits of the remote server accordingly: per-user, per-IP and global limits need room for the operations a session runs in parallel. Clients commonly request metadata while a transfer is in progress, so a session downloading a single file can already hold two connections. An operation that does not obtain a connection fails with the error the server returned.

The connect phase and each command on the control connection are bounded by a 20 second timeout.

To prevent connection loops, SFTPGo detects self-connections (an FTP backend pointing back to the same SFTPGo instance) and rejects them unless `allow_self_connections` is explicitly enabled in the [configuration](config-file.md).
