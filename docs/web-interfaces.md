---
description: "SFTPGo web interfaces: WebAdmin for centralized server management and WebClient for browser-based file management, sharing, and 2FA."
---

# Web User Interfaces

SFTPGo provides two web interfaces, both available with dark and light themes.

## WebAdmin

The WebAdmin UI allows administrators to create and manage users, groups, virtual folders, event rules, and other server resources.

![Initial Screen](assets/img/initial-screen.png){data-gallery="initial screen"}

With the default configuration, it is available at:

[http://127.0.0.1:8080/web/admin](http://127.0.0.1:8080/web/admin){:target="_blank"}

On first access (when no admin exists in the data provider), SFTPGo presents a setup screen to create the initial admin account. You can also pre-create an admin by loading initial data or by setting the environment variables `SFTPGO_DEFAULT_ADMIN_USERNAME` and `SFTPGO_DEFAULT_ADMIN_PASSWORD` (with `create_default_admin` enabled in the data provider configuration).

## WebClient

The WebClient UI gives end users a browser-based file manager. Users can browse and manage their files, change credentials, and configure two-factor authentication.

![WebClient files](assets/img/web-client-files.png){data-gallery="client-files"}

With the default configuration, it is available at:

[http://127.0.0.1:8080/web/client](http://127.0.0.1:8080/web/client){:target="_blank"}

Key capabilities:

- **File management** — Upload, download, rename, delete, and create directories. Multiple files or folders can be downloaded as a single ZIP archive (non-regular files such as symlinks are silently skipped).
- **File integrity** — Compute the SHA-256 hash of a file on demand, copy it, or download a checksum list verifiable with standard tools. See [File integrity](#file-integrity) below.
- **Sharing** — Create HTTP/S links to share files and folders externally, with optional password or email-based authentication, IP restrictions, download/upload limits, and automatic expiration.
- **Credentials** — Change password and manage public keys (public key management can be disabled per-user via permissions).
- **Two-factor authentication** — Set up TOTP-based 2FA, compatible with Microsoft Authenticator, Google Authenticator, Authy, and similar apps.

### Disabling the WebClient

The WebClient can be disabled:

- **Globally** — Set `enable_web_client` to `false` in the `httpd` binding configuration.
- **Per-user** — Add `HTTP` to the user's denied protocols list.

### File integrity

Compute the SHA-256 hash of stored files on demand, on any storage backend. For encrypted storage the hash is taken on the file content as the user sees it.

- **Single file** — Choose **Compute hash** from the file actions menu. The dialog shows the hash; copy it or download it as `SHA256SUMS.txt`.
- **Multiple files** — Select files and choose **Compute hash** from the bulk actions menu to download one `SHA256SUMS.txt` covering the selection.

`SHA256SUMS.txt` is in the standard GNU coreutils format and verifies with common tools:

```shell
sha256sum -c SHA256SUMS.txt
```

`shasum -c` and `rclone --checkfile` accept it as well. The single-file **Copy** button copies the same `hash  filename` line (hash, two spaces, name), so the clipboard value is directly checkable. Use it to confirm a file after an upload or download, to give a recipient a checksum, or to check a set of files against a local manifest.

:information_source: Computing a hash reads the whole file, so it counts as a download: it uses the user's download quota and bandwidth limits, requires the `download` permission, is written to the transfer log, and triggers `download` event rules — once per file in a multi-file selection.

:warning: On cloud storage the read is billable egress. Cancelling a running computation stops the server-side read; bytes already read are still billed.

## Security

### Authentication

Both interfaces support:

- **Username and password** login.
- **OpenID Connect (SSO)** — Integrate with external identity providers (Microsoft Entra ID, Google, Keycloak, Okta, etc.). See [OpenID Connect](oidc.md).
- **Two-factor authentication** — TOTP-based, configurable for both admin and user accounts.

Login methods can be selectively enabled or disabled per binding via the `disabled_login_methods` configuration parameter.

### HTTPS and mutual TLS

The web interfaces can be served over HTTPS. For each binding, you can:

- Enable HTTPS by providing a certificate and key file.
- Require mutual TLS (client certificate authentication) in addition to standard credentials.
- Configure the minimum TLS version and allowed cipher suites.

Certificates can be automatically managed via the built-in [ACME protocol](tutorials/lets-encrypt-certificate.md) (Let's Encrypt).

### Content Security Policy

SFTPGo supports strict Content Security Policies — neither `unsafe-eval` nor `unsafe-inline` are required. Configure CSP and other security headers in the `security` section of the `httpd` configuration.

### Reverse proxy

When running behind a reverse proxy, configure `proxy_allowed` and `client_ip_proxy_header` on the binding to ensure SFTPGo sees the real client IP addresses. This is important for the defender, rate limiting, and audit logging.

## Internationalization

SFTPGo uses the [i18next](https://www.i18next.com/){:target="_blank"} framework for internationalization. The following languages are supported: English, Italian, German, French, Spanish, and Chinese (Simplified).

Translations are managed via [Crowdin](https://crowdin.com/project/sftpgo){:target="_blank"}.
