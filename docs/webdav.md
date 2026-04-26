---
description: "SFTPGo WebDAV server configuration: HTTPS, CORS, file locking, and client compatibility (Windows, macOS, Linux)."
---

# WebDAV

SFTPGo includes a WebDAV server that can be enabled by configuring one or more bindings in the `webdavd` [configuration](config-file.md) section. WebDAV provides HTTP-based file access, making it compatible with a wide range of clients and operating systems.

Each user accesses their home directory at `http(s)://<host>:<port>/<prefix>`. The default prefix is empty (resources at `/`); a custom prefix can be set per-binding (e.g., `/dav`).

## Authentication

WebDAV supports the same authentication methods as FTP:

- **Password** — HTTP Basic authentication.
- **TLS certificate** — Mutual TLS with client certificate validation.
- **Certificate + password** — Combined authentication.

### User caching

Unlike SFTP and FTP, WebDAV has no persistent session — each HTTP request is independently authenticated. To avoid repeated database queries and password hash computations, SFTPGo caches authenticated users in memory.

The cache is configured in the `webdavd` section:

- **Expiration time** — How long (in minutes) a cached user remains valid. After expiration, the next request triggers a fresh database query. `0` means no expiration. Note: while a user is cached, `last_login` is not updated and `external_auth_hook`, `pre_login_hook`, and `check_password_hook` are not executed.
- **Max size** — Maximum number of cached users. When the limit is reached, the oldest entry is evicted. `0` means no limit (capped by the total number of users).

Users are automatically removed from the cache on update or delete.

## MIME type detection

WebDAV requires a MIME type for each file. SFTPGo uses the following detection strategy:

1. **Extension-based** — Guesses the MIME type from the file extension.
2. **HEAD request** — For cloud storage backends, issues a HEAD request to retrieve the content type.
3. **Content sniffing** — As a last resort, reads the first 512 bytes to detect the type.

Steps 2 and 3 may slow down directory listings for directories with many files having unregistered extensions. To mitigate this, enable **MIME type caching** — once detected, the MIME type is cached in memory and reused. You can also add custom extension-to-MIME mappings in the configuration, or register them at the OS level (`/etc/mime.types` on Linux, the registry on Windows).

## Lock support

SFTPGo implements WebDAV locking (LOCK/UNLOCK methods) with an in-memory lock manager. Exclusive write locks are supported. Each authenticated user gets a dedicated lock manager instance, and locks are properly cleaned up when resources are deleted or renamed.

## Reverse proxy

When running WebDAV behind a reverse proxy:

- Configure `proxy_allowed` and `client_ip_proxy_header` on the binding to ensure SFTPGo sees real client IP addresses.
- **Preserve the `Host` header** — WebDAV `COPY` and `MOVE` operations will fail if the Host header is rewritten. For Apache, set `ProxyPreserveHost On`.
- Alternatively, set `proxy_mode` to `1` to use the PROXY protocol instead of HTTP headers.

## CORS

CORS (Cross-Origin Resource Sharing) can be configured per-binding for browser-based WebDAV clients. The configuration is in the `cors` section of each `webdavd` binding. See the [configuration reference](config-file.md) for all available options.

## Known issues and limitations

- **Directory removal on cloud backends** — Removing a directory tree may produce a "not found" error when deleting the last (virtual) directory, if the client removes files and directories individually instead of issuing a single remove command.
- **Permission requirements** — Listing a directory requires both `list` and `download` permissions. Uploading files requires both `list` and `upload` permissions.
- **Error handling in listings** — If a file or directory is inaccessible (e.g., OS permissions, missing virtual folder path), it is silently omitted from the listing. A different error causes the entire listing to fail. This differs from SFTP/FTP, where inaccessible entries still appear in listings.
- **Dead properties** — SFTPGo has a minimal implementation: modification time can be set and is returned in live properties, but arbitrary dead properties are not persisted.
- **PROPFIND Depth** — `Depth: 0` and `Depth: 1` are supported. `Depth: infinity` is not allowed.

### Windows native client

The Windows WebDAV redirector has specific limitations. Please review the [registry settings](https://docs.microsoft.com/en-us/iis/publish/using-webdav/using-the-webdav-redirector#webdav-redirector-registry-settings){:target="_blank"} carefully:

- The default file size limit is **50 MB** — increase `FileSizeLimitInBytes` if needed.
- If SFTPGo is not configured with HTTPS, set `BasicAuthLevel` to `2` to allow Basic authentication over HTTP.
