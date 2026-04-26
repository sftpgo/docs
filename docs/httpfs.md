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

The password and API key are stored encrypted according to your [KMS configuration](kms.md).

### Unix domain socket support

The endpoint can point to a Unix domain socket instead of a TCP address:

```shell
http://unix?socket_path=/path/to/socket&api_prefix=/api
https://unix?socket_path=/path/to/socket&api_prefix=/api
```

The `socket_path` parameter must be an absolute path. The `api_prefix` is prepended to all request paths.

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
