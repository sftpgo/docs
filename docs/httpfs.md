---
description: "Use a custom HTTP/S service as an SFTPGo storage backend via a simple REST API contract."
---

# HTTP as storage backend

SFTPGo can use a custom HTTP/S service as a storage backend. This allows you to integrate SFTPGo with any storage system by implementing a simple REST API.

:warning: The HTTP filesystem backend is a work in progress. The API interface may change in a backward-incompatible way in future releases.

## Configuration

| Parameter | Description |
| ----------- | ------------- |
| **Endpoint** | Base URL of the HTTP backend (e.g., `https://storage.example.com/api`). Required. |
| **Username** | Username for HTTP Basic authentication. Optional. |
| **Password** | Password for HTTP Basic authentication. Optional. |
| **API key** | API key sent via the `X-API-KEY` header. Optional. Can be used in addition to or instead of Basic authentication. |
| **Equality check mode** | How to determine if two configurations point to the same server: `0` (default) requires only the endpoint to match; `1` also requires the username to match. Used for rename operations across configurations. |
| **Skip TLS verify** | Accept any TLS certificate. :warning: Only for testing — susceptible to man-in-the-middle attacks. |
| **Remote directory** | Optional server-side path used as the starting directory for all operations. See [Remote directory](#remote-directory) below. |

The password and API key are stored encrypted according to your [KMS configuration](kms.md).

### Unix domain socket support

The endpoint can point to a Unix domain socket instead of a TCP address:

```shell
http://unix?socket_path=/path/to/socket&api_prefix=/api
https://unix?socket_path=/path/to/socket&api_prefix=/api
```

The `socket_path` parameter must be an absolute path. The `api_prefix` is prepended to all request paths.

## Remote directory

By default, operations are relative to the resource root of the HTTP backend. Set a **Remote directory** to scope the backend to a sub-path, so users land directly in the right place: SFTPGo prepends it to the resource path of every request (the `{name}` segment). This is independent of the `api_prefix` endpoint option, which prefixes the HTTP route rather than the resource path; the two compose.

:warning: The remote directory is **not a security boundary**. The HTTP backend is opaque to SFTPGo: if the server behind it is backed by a local filesystem, a server-side symlink under the remote directory may resolve outside it, and SFTPGo has no protocol-level way to detect this. Treat the remote directory as a convenience starting path, not a confinement.

Because of this, the feature is opt-in at the deployment level: the `http` backend must be listed in the [`allow_remote_directory`](config-file.md) setting of the `common` configuration section. The remote directory is honored only while the backend is enabled — a connection using a stored configuration whose backend is not in the allow list is rejected, with the reason logged. The value is preserved on save and on backup restore, so disabling the backend does not break data import; in the WebAdmin the field is shown when the backend is enabled, or with a warning when a value is set while the backend is disabled, so it can be cleared.

When the backend is defined at the group level, the remote directory supports the same placeholders as the cloud key prefix and the SFTP prefix (for example `%username%`, `%role%`, `%customN%`). The placeholders are resolved per user when the group settings are applied, so a single group configuration can scope each member to their own sub-path.

## API contract

The HTTP backend must implement a set of REST endpoints. SFTPGo appends the operation path to the configured endpoint URL. The [OpenAPI schema](https://github.com/drakkan/sftpgo/blob/main/openapi/httpfs.yaml){:target="_blank"} defines the full API contract.

Key endpoints:

| Method | Path | Description |
| -------- | ------ | ------------- |
| GET | `/stat/{name}` | File/directory metadata |
| GET | `/open/{name}?offset={offset}` | Download file (with optional offset) |
| POST | `/create/{name}` | Upload file |
| PATCH | `/rename/{name}?target={target}` | Rename/move |
| DELETE | `/remove/{name}` | Delete |
| POST | `/mkdir/{name}` | Create directory |
| PATCH | `/chmod/{name}?mode={mode}` | Change permissions |
| PATCH | `/chtimes/{name}` | Change timestamps |
| PATCH | `/truncate/{name}?size={size}` | Truncate file |
| GET | `/readdir/{name}` | List directory |
| GET | `/dirsize/{name}` | Directory size |
| GET | `/statvfs/{name}` | Filesystem stats |

SFTPGo maps HTTP status codes to filesystem errors: `404` = file not found, `501` = not supported, `403` = permission denied, etc.

## Limitations

- Upload resume is not supported.
- Atomic uploads are not supported.
- Opening a file for both reading and writing at the same time is not supported.
- `chown` and `symlink` are not supported.
- Virtual folders are not supported on this backend.
