---
description: "Release notes and changelog for SFTPGo Enterprise — new features, improvements, and bug fixes for each version."
---

# Release Notes

This page provides a concise overview of the new features, improvements and bug fixes introduced in each SFTPGo Enterprise release.
We encourage you to check back regularly to stay up to date with the latest changes and to make the most of all enhancements.

## Compatibility notes

Upgrading to the Enterprise edition of SFTPGo is supported starting from Open Source release 2.6.x.

If you're migrating from an open-source installation, please follow the guide here: [**Migration from Open-Source to Enterprise Edition**](tutorials/migrating.md)

## Update June 26, 2026 - v2.7.20260626

### New features

- **FIPS 140-3 mode**: dedicated [FIPS builds](fips.md) for Linux (`sftpgo-fips`/`sftpgo-plugins-fips`), [Windows](https://download.sftpgo.com/windows/sftpgo_windows_fips_x86_64.exe), and Docker (`-fips` image variants), compiled against a FIPS-validated cryptographic module. Part of the [Ultimate tier](https://sftpgo.com/on-premises); a FIPS build requires a license with the FIPS feature enabled. See [FIPS 140-3 mode](fips.md).
- Bandwidth limits: per-user upload and download caps are now enforced as an aggregate across all of a user's concurrent transfers on an instance, instead of per transfer, so parallel transfers across handles or protocols stay within the configured rate. Per-source overrides aggregate within their own source group.
- Azure Blob Storage: the [managed identity client ID](azure-blob-storage.md#selecting-a-managed-identity) can be set per storage, selecting which user-assigned identity SFTPGo authenticates with when the default Azure credential chain exposes more than one. Each user or virtual folder can target a different identity.
- Password hashing: [PBKDF2-SHA256](password.md) is now selectable for hashing new passwords, alongside bcrypt and argon2id, with configurable iterations and salt length. Passwords stored in another supported format — including admin and share passwords — are transparently re-hashed to the configured algorithm at the next successful login.
- Storage backends: the [FTP](ftpfs.md#remote-directory) and [HTTP](httpfs.md#remote-directory) backends can be scoped to a server-side **remote directory**, so users land in a sub-path of the remote tree. This is not a security boundary and is opt-in per backend via the new [`allow_remote_directory`](config-file.md) setting in the `common` section. Disabled by default.
- SFTP backend: the new `disable_concurrent_writes` option forwards each upload write request to the remote server sequentially. Enable it for legacy servers that do not support SFTP pipelining.
- LDAP authentication: a [role group prefix](plugins/ldap-auth.md#mapping-a-role) can map LDAP group membership to an SFTPGo role, enabling delegated multi-tenant administration. The matched role must already exist, and a user matching more than one role group is denied access.
- KMS secret encryption: the new [`convertsecrets`](cli.md#re-encrypting-the-stored-secrets) command re-encrypts every stored secret onto a new master key and/or provider in a single pass, so you can rotate the KMS master key or migrate provider without losing access to the data. On a networked SQL database a [master key ring](config-file.md#master-key-ring) lets the running server decrypt the old and new key at once, for a rotation with no downtime.
- Internationalization: added a [Hungarian](web-interfaces.md#internationalization) translation for the WebAdmin and WebClient UIs.
- Symbolic links: the new [`symlink_mode`](config-file.md#symbolic-links-and-permissions) setting selects, per backend, whether clients holding the `create_symlinks` permission may create symbolic links — on the local filesystem, the SFTP backend, or both. It is disabled by default. Creating a link requires `create_symlinks` on both the link's directory and the directory it points into, so per-directory permissions are enforced consistently on the path the client requests.

### Bug fixes

- Event Manager: fixed a bug in the filesystem [Copy](filesystem-actions.md#copy), [PGP](filesystem-actions.md#pgp) and [Delete](filesystem-actions.md#delete) actions where, in some edge cases, a path could be incorrectly identified as a wildcard pattern, causing the action to silently match nothing.
- SFTP: fixed a regression introduced in `v2.7.20260528` where a client opening multiple sessions on the same SSH connection could fail with a "user filesystem cache has been closed" error when one of the sessions ended.
- Local filesystem: fixed an edge case in path resolution on Windows shares.

### Backward incompatible changes

- Event Manager: a filesystem [Copy](filesystem-actions.md#copy), [PGP](filesystem-actions.md#pgp) or [Delete](filesystem-actions.md#delete) action whose last path component combines a wildcard with a placeholder is now rejected on save or import. Existing rules of that shape keep working at runtime but must be rewritten on the next edit or data import.
- KMS secret encryption: secrets sealed by the `builtin://` provider of open-source SFTPGo 1.2.x or earlier (deprecated since 2.0.0) now fail to decrypt with an error. Re-enter those credentials as plain text to re-encrypt them with the [current scheme](kms.md).
- Cloud storage key prefix: the `key_prefix` of the [S3](s3.md), [Google Cloud Storage](google-cloud-storage.md) and [Azure Blob](azure-blob-storage.md) backends is now normalized consistently across the REST API, data import and the WebAdmin. A non-empty value that resolves to the storage root is now rejected. Existing configurations keep working at runtime; the check applies the next time one is saved or imported.
- Symbolic links: creating symbolic links is now disabled by default. In earlier releases a client holding the `create_symlinks` permission could always create links; creation must now be enabled per backend with [`symlink_mode`](config-file.md#symbolic-links-and-permissions). In addition, creating a link now requires `create_symlinks` on both the link's directory and the directory it points into.

### Migration notes

- KMS secret encryption: new secrets are now sealed with [AES-256-GCM](kms.md) instead of NaCl secretbox. This is transparent on upgrade (each secret records its own scheme), but an older release cannot read the new format. Before **downgrading**, re-enter affected secrets as plain text on the old binary, or use [`convertsecrets`](cli.md#re-encrypting-the-stored-secrets) to migrate them back first.
- Bandwidth limits: per-user caps now bound a user's combined transfer rate on a single instance, not each transfer. If you sized them expecting every transfer to reach the full rate, raise the limits for users that rely on concurrent transfers.
- Symbolic links: creating symbolic links is disabled by default after upgrade. If any workflow relies on clients creating them, set [`symlink_mode`](config-file.md#symbolic-links-and-permissions) to enable the backends that need it: `1` for the local filesystem, `2` for the SFTP backend, `3` for both. Symbolic links already present on the storage continue to be followed regardless of this setting.

### Security fixes

- Folder Permission Bypass via Symlink Dereference (severity: Moderate). [CVE-2026-10031](https://github.com/drakkan/sftpgo/security/advisories/GHSA-fj9v-mxr3-w75w).

## Update May 28, 2026 - v2.7.20260528

### New features

- Shares: Added [allowed share paths](tutorials/shares.md#restricting-shareable-paths) filter, complementing the existing denied paths list. Administrators can define an allowlist of virtual paths within which shares may be created; paths outside the allowlist are rejected. The allowed and denied lists work together using longest-prefix matching, so a broad allowlist can be paired with narrower denied entries to carve out exceptions.
- Event Manager: Data retention check now supports [folder-scoped execution](tutorials/eventmanager-retention.md#folder-scoped-retention) and [dry-run mode](tutorials/eventmanager-retention.md#dry-run). Folder-scoped retention runs once on a selected virtual folder as a system task; dry-run produces the report without deleting files or creating archive copies.
- Event Manager: Enhanced the [PGP](filesystem-actions.md#pgp) filesystem action with glob patterns, per-entry source disposition, and directory-style targets. Source paths can use wildcards in the last component (e.g., `/inbox/*.csv`) to encrypt or decrypt multiple files in one entry. Each entry has its own **After encrypt/decrypt** option (Keep, Delete, Move) so different sources in the same action can have different post-processing. When the source is a single file, a target ending with `/` is treated as a destination directory and the output filename is derived from the source (`report.csv` → `report.csv.pgp` for encrypt, `report.csv.pgp` → `report.csv` for decrypt). See the [PGP tutorial](tutorials/eventmanager-pgp.md#example-scheduled-batch-encryption-with-wildcards) for examples.
- Event Manager: The filesystem [Delete](filesystem-actions.md#delete) action now supports glob patterns in the last path component (e.g., `/inbox/*.tmp`) to remove multiple matching entries in a single action.
- Admin permissions: Introduced [granular catalog permissions](admin-permissions.md) for groups and folders, with three independent grants per catalog (`view_*`, `manage_*`, `del_*`) mirroring the user permission model. The change brings:
  - **Read-only detail pages**: admins with `view_users` / `view_groups` / `view_folders` can open the corresponding detail page without needing the edit permission; the page renders read-only and inspection-friendly.
  - **Auto-apply of `Admin.Groups` on user create**: an admin without `view_groups` automatically gets the configured admin groups attached to every new user.
  - **Cross-permission rule**: granting `view_groups` or `manage_groups` also requires `view_folders`, since a group response embeds its folder mappings and the WebAdmin group form lists folder names.
  - **Legacy compatibility env var**: `SFTPGO_HOOK__LEGACY_ADMIN_PERMS=1` restores the previous umbrella semantics and bypasses the cross-permission rule, for in-place upgrades that need a transition bridge.
- WebClient: Added on-demand [file integrity](web-interfaces.md#file-integrity) checks. Compute the SHA-256 hash of a stored file on any backend, copy it, or download a `SHA256SUMS.txt` for one or many files that verifies with `sha256sum -c`, `shasum -c`, and `rclone --checkfile`.
- Data provider: Added the [`convertnames` command](cli.md#converting-names-to-the-configured-naming-rules) to reconcile existing usernames, folder, group, role, event action and event rule names with the configured `naming_rules`. It reports the names to convert and any blocking collision without modifying data; `--execute` renames them with a single statement that changes only the name column, and is idempotent and re-runnable. Conversion is available for the SQL providers; on bolt the report is produced.
- SSH: Added two new [multi-step authentication](ssh.md#multi-step-authentication) combinations — `password+publickey` and `keyboard-interactive+publickey` — complementing the existing `publickey+password` and `publickey+keyboard-interactive`. The first listed factor is the one the client must offer first; each combination is a separate entry in `denied_login_methods`.
- SSH: Added support for the [`uname` command](ssh.md#ssh-commands). SFTPGo does not execute the system `uname` binary; it returns a fixed `uname -a`-style string and supports the standard field-selector flags. Useful for backup tools and other clients that probe the server with `uname` over SSH. The base string defaults to `Linux sftpgo 1.0.0 #1 SMP SFTPGo x86_64 GNU/Linux` and can be customized via the [`SFTPGO_HOOK__SSHD_UNAME_OUTPUT`](env-vars.md#additional-environment-variables) environment variable.
- Defender: Added [SQLite](defender.md) as a supported data provider for the `provider` defender driver, in addition to MySQL, PostgreSQL and CockroachDB. Single-instance deployments using the SQLite data provider can now persist host scores and bans across restarts without switching to a shared database.

### Hardening

- Event Manager: The synthetic user used by Backup chains is restricted to list and download permissions. Downstream actions can attach the dump file to email or HTTP notifications but cannot write to or delete files inside the backups directory, including the dump itself.
- Data At Rest Encryption / S3 SSE-C: the CryptFs passphrase and the S3 SSE-C key are validated against a configurable minimum entropy ([`common.secret_min_entropy`](config-file.md), default 80) when set or changed. These secrets are key material for at-rest encryption; a weak value could be brute-forced offline from leaked ciphertext. The check applies only to plain-text submissions, so secrets already stored are unaffected. See [Passphrase strength](dare.md#passphrase-strength).

### Bug fixes

- Defender IP list: Fixed match resolution when an IP is covered by multiple entries with conflicting modes. The most specific entry (longest network prefix) now wins, so a narrow allow correctly overrides a broader deny and vice versa. Previously the broader entry could be returned, making narrow exceptions to a `0.0.0.0/0` (or `::/0`) deny ineffective. The allow list, rate-limiter safe list and trusted list are unaffected (they reject deny entries at validation time).
- Connection admission no longer rejects a new connection when the user's active transfers are saturated. Only the concurrent session count is checked when a connection is established; the transfer limit is enforced when each transfer starts. A client within its `max_sessions` budget can open an additional connection (for example to browse) while transfers are in progress.
- Shares: the `max_tokens` usage limit is now enforced atomically. A race between concurrent share accesses could let the number of downloads/uploads exceed the configured maximum; the usage counter is now reserved with a conditional update so the limit holds under concurrency.
- Transfer admission: `max_total_transfers`, `max_per_host_transfers` and the per-user `max_sessions` cap are now strictly enforced. Concurrent bursts could previously let the number of active transfers exceed the configured cap; the limits are now applied atomically. Denials surface via the new [`sftpgo_transfer_admission_denied_total`](metrics.md#transfer-admission) metric.
- Connection admission: `max_per_host_connections` and `max_total_connections` are now strictly enforced under concurrent connection bursts. Denials surface via the new [`sftpgo_connection_admission_denied_total`](metrics.md#connection-admission) metric. HTTP and WebDAV now return `429 Too Many Requests` for cap-driven denials (previously `403 Forbidden` on HTTP and `503 Service Unavailable` on WebDAV); allow-list denials on WebDAV change from `503` to `403`. The `post_connect_hook` no longer fires for connections denied by the cap or allow list.

### Security fixes

- Path confinement bypass in the partial ZIP download of browsable shares (severity: Moderate). [CVE-2026-49244](https://github.com/drakkan/sftpgo/security/advisories/GHSA-h64p-8h4r-6gfh).
- Stored XSS via the `inline` parameter on file download endpoints (severity: Low). [CVE-2026-49245](https://github.com/drakkan/sftpgo/security/advisories/GHSA-3vcg-pv95-pq54).

### Migration notes

- The `manage_groups` and `manage_folders` admin permissions now grant add and edit only. Listing requires `view_groups` / `view_folders`; deletion requires `del_groups` / `del_folders`. Administrators that previously relied on the umbrella semantics keep working by setting `SFTPGO_HOOK__LEGACY_ADMIN_PERMS=1`, or by granting the explicit permissions to existing admins. See [Admin Permissions](admin-permissions.md#legacy-compatibility-env-var) for the recommended migration steps.
- Deleting a virtual folder, role or group that is still referenced is now rejected; detach it from every referencing user, group or administrator first. The previous behavior silently cleared the reference, which could revoke a mounted path, lift role-based scoping, or stop an admin group's auto-apply on user creation without warning.
- Updating a group or virtual folder no longer fans out synthetic `user.update` provider events to the affected users. Event Manager rules and notifier plugins now receive the corresponding `group.update` / `folder.update` event only; subscribe to those if you also need to react to indirect changes on a user's effective configuration.
- [`common.secret_min_entropy`](config-file.md). Creating or updating a user or folder with a CryptFs passphrase or S3 SSE-C key supplied in plain text (REST API, Terraform, `loaddata`, WebAdmin) now requires that secret to meet an entropy threshold. Secrets already stored are not re-validated and keep working. To attach to data encrypted with a legacy weak secret, lower or set `common.secret_min_entropy` to `0`; see [Passphrase strength](dare.md#passphrase-strength).
- The new SSH multi-step combinations `password+publickey` and `keyboard-interactive+publickey` are enabled by default, mirroring the historical defaults of `publickey+password` and `publickey+keyboard-interactive`. Existing users do not have these entries in their `denied_login_methods` filter, so after the upgrade they can authenticate in either ordering: a user previously locked to "publickey then password" can now also complete the flow as "password then publickey" (and as "keyboard-interactive then publickey" when the keyboard-interactive hook is configured). Both factors are still required to log in, so this is an expansion of accepted flows rather than a relaxation of credentials. To restrict the user back to a single ordering, add the unwanted combinations explicitly to `denied_login_methods` — for example `["publickey", "password", "keyboard-interactive", "publickey+keyboard-interactive", "password+publickey", "keyboard-interactive+publickey"]` to force the publickey-first / password-second flow exclusively.

## Update April 26, 2026 - v2.7.20260426

### New features

- Event Manager: Added Event Report action. Generates aggregated reports of filesystem events grouped by user via the eventsearcher plugin. Supports configurable time windows, action/status filters, and per-user or aggregated delivery. The generated reports can be used in subsequent actions, for example to send email notifications with CSV attachments or to post structured data to HTTP webhooks.
- Event Manager: Enhanced the filesystem Copy action with source disposition, glob patterns, retry, and continue-on-error support:
  - **Source disposition**: optionally delete or move source files after a successful copy, per entry. Move supports both same-filesystem (atomic rename) and cross-filesystem (streaming copy + delete) targets. Eliminates race conditions in copy-then-delete workflows.
  - **Glob patterns**: source paths can use wildcards in the last component (e.g., `/inbox/*.csv`). Only matching files are processed.
  - **Per-file retry**: configurable retry count (0–5) with a fixed 5-second delay. Applied independently to each file copy and disposition operation.
  - **Continue on error**: when enabled, the action processes all files even if some fail, collecting errors for reporting at the end.
- Event Manager: The filesystem Delete action now supports trailing `/` to delete directory contents without removing the directory itself (e.g., `/inbox/` removes all files and subdirectories inside `/inbox` but keeps the directory).
- Event Manager: Added `fromNanos` helper function to easily convert Unix timestamp expressed in nanoseconds into time objects.
- Event Manager: Added `Initiator` template variable for provider events. Exposes the full object (User or Admin) of the entity that initiated the action.
- Event Manager: Added [Execute before file publish](./execute-before-file-publish.md) option for upload event actions. Runs actions on the temporary atomic upload file before it becomes visible to other users. Supports both asynchronous mode (client receives immediate OK) and synchronous mode (client waits for action result). Designed for antivirus scanning (ICAP), content validation, DLP checks, and other pre-publication processing. Works on all storage backends.
- WebClient: Added `shares-policy-change-disabled` option. When set (on user or inherited from group), hides the share group policy UI and ensures only enforced policies are applied, preventing users from adding or modifying group bindings on their shares.
- WebAdmin: Added password expiration column to the users list. Displays the expiration date calculated from the user's last password change and the configured expiration policy.
- Shares: Added [denied share paths](tutorials/shares.md#restricting-shareable-paths) filter. Administrators can define virtual paths that users are not allowed to share, configurable per-user or inherited from groups.
- Shares: Added [share restriction filters](tutorials/shares.md#restricting-shareable-paths) for paths and scopes. Administrators can define virtual paths that users are not allowed to share, and restrict which share scopes (read, write, read/write) are available. Both settings are configurable per-user or inherited from groups. When all scopes are denied, sharing is completely disabled.
- Event Manager: Path filter conditions can now optionally [match on the filesystem path](eventmanager.md#path-filters) instead of the virtual path. This allows filtering by physical storage location, bucket name, Windows drive letter, or UNC path. Each pattern can independently target the virtual or filesystem path. Backslashes in patterns are auto-normalized to forward slashes. Matching is case-sensitive on all platforms.
- WebAdmin: Added language configuration section to the global configs page. Allows administrators to select which UI languages are enabled system-wide. When configured, overrides per-binding language settings across all instances.
- WebAdmin: Added TLS certificate configuration section to the global configs page. Allows administrators to upload a TLS certificate and private key directly from the UI, with per-protocol selection (HTTPS, FTPS, WebDAV). The certificate replaces the global file-based certificate for the selected protocols. Existing certificates can be updated without a service restart. Mutually exclusive with automatic certificates (Let's Encrypt/ACME).
- Automatic Certificates (ACME): Certificates, account keys, and registration data are now stored in the database instead of on disk. This enables stateless deployments (containers) and proper cluster operation. Existing on-disk data is automatically migrated to the database on the next certificate renewal check. Certificate renewal uses database-level optimistic locking for cluster-safe coordination, replacing the previous file-based lock.
- Event Manager: Improved scheduled rule concurrency guard. Rules with `GuardFromConcurrentExecution` now release the task lock immediately after completion, allowing other nodes to execute the rule without waiting for the timeout period.
- WebAdmin: Added [email templates](email-templates.md) configuration section to the global configs page. Allows administrators to customize the subject and HTML body of system-sent emails (password reset, password expiration, one-time access codes) directly from the UI, with a visual HTML editor. Both subject and body support Go template placeholders (e.g., `{{.Username}}`, `{{.Code}}`). When configured, overrides the built-in default templates across all instances.
- HTTPD: Added `disable_auto_tls` option for bindings. When set, automatic TLS certificates (from the WebAdmin TLS configuration or ACME) will not be applied to the binding. Useful for bindings dedicated to intra-node communication in cluster setups.
- HTTP client: Added `skip_tls_verify_for_urls` configuration option. Allows disabling TLS certificate verification for specific URL prefixes instead of globally.
- OIDC: Added support for PKCE-only authentication without a client secret, enabling integration with public OAuth2 clients. When the client secret is omitted, SFTPGo uses PKCE exclusively to secure the authorization code exchange.
- Metrics: Added per-user Prometheus metrics — uploads, downloads, bandwidth, active connections, and login success/failure tracked per username. Always enabled when the telemetry server is active. See the [metrics documentation](metrics.md) for details.
- Metrics: Added filesystem operation metrics (`sftpgo_fs_ops_total`) tracking rename, delete, rmdir, copy, and mkdir operations across all protocols and event manager actions.
- Metrics: Added per-backend transfer and metadata operation metrics with label-based filtering. See the [metrics documentation](metrics.md) for the complete list.
- S3: Added configurable `checksum_algorithm` (CRC32, CRC32C, CRC64NVME, SHA1, SHA256) for upload integrity verification. When set, checksums are computed and sent with uploads and copies so S3 can verify payload integrity. Empty by default for maximum compatibility with S3-compatible services.
- Azure Blob: Uploads now send a CRC64 transactional checksum for server-side verification of each block. Can be disabled via the `SFTPGO_HOOK__AZBLOB__DISABLE_CHECKSUM` environment variable.
- WebAdmin: Added a "Test connection" button in the filesystem configuration section of user, group, and folder edit pages. Verifies connectivity, authentication, and configured-path access for remote backends (S3, Google Cloud Storage, Azure Blob, SFTP, FTP, HTTP) without transferring files.

### Bug fixes

- Event Manager: Fixed execution of source-only folder actions (delete, exist, mkdir, rename, metadata check). When a source virtual folder is configured, these actions now correctly execute once instead of per-user.
- Event Manager: Fixed provider event role filtering. Role-restricted admins now see events for their tenant's users and admins even when the event is triggered by a global admin; the `role` field on provider events now identifies the affected object's tenant instead of the executor's role.
- HTTP client: Fixed URL prefix matching for headers and TLS verify exceptions. Matching now requires a boundary character (`/`, `?`, `#`) or exact match. Default ports are normalized (`https://host:443` is equivalent to `https://host`).
- TUS: Fixed the `Location` header in upload creation responses to respect `X-Forwarded-Proto` from trusted proxies. When running behind a TLS-terminating reverse proxy (e.g., Cloudflare), the header used `http://` instead of `https://`, causing browsers to block subsequent upload requests as mixed content.

### Breaking changes

Overhauled Prometheus metrics. This is a **breaking change** for existing dashboards and alerts — all metric names have been updated. See the [metrics documentation](metrics.md) for the complete list of new metric names.

- Renamed byte counters to follow Prometheus conventions: `sftpgo_upload_size` → `sftpgo_upload_size_bytes` (same for all download/backend size metrics).
- Replaced 24 individual login counters with a single `sftpgo_login_total{method, result}` CounterVec.
- Replaced per-backend counters (S3, GCS, Azure, SFTPFs, HTTPFs) with `sftpgo_backend_*{backend}` CounterVec metrics. Added FTPFs backend tracking.
- Replaced per-backend metadata operation counters with `sftpgo_backend_ops_total{backend, operation}` and `sftpgo_backend_ops_errors_total{backend, operation}`.
- Removed SSH command metrics (`sftpgo_ssh_commands_total`, `sftpgo_ssh_command_errors_total`).
- Removed HTTP request metrics (`sftpgo_http_req_total`, `sftpgo_http_req_ok_total`, `sftpgo_http_client_errors_total`, `sftpgo_http_server_errors_total`).

Provider event notifications (plugin notifier, HTTP hook, command hook) now report the `role` of the affected object (user's or admin's tenant) instead of the executing admin's role. This aligns with the Event Manager `role_names` filter and role-scoped provider event search. External consumers that relied on the previous executor-role semantic will see a different value.

## Update March 7, 2026 - v2.7.20260307

### New features

- WebClient/REST API: Added support for the TUS resumable upload protocol, enabling chunked, resumable uploads and improving compatibility with services such as Cloudflare and other proxy/CDN environments.
- WebAdmin: Added a dedicated API Keys section to create and manage API keys directly from the UI. Previously, they could only be managed via the REST API.
- OIDC: Added role mapping for SFTPGo users, with full support for overlapping Admin and User roles in the Web UI. Fully backward compatible with existing setups.
- OIDC: added support for explicitly specifying the token issuer URL for verifying received ID tokens, useful for certain off-spec OpenID Connect providers.
- WebUI: Refactored the HTML editor to improve maintainability and security.
- WebClient: Added a diff viewer in the editor to compare different file versions.
- REST API: Added support for retrieving file checksums.
- SFTPD: Added support for `SSH_FXP_STAT/FSTAT` calls on ongoing uploads to atomic storage backends, where files remain invisible until the upload completes. Applies to requests from the same connection that initiated the upload.
- SFTPD: Added banners to inform users of fatal error conditions.
- SFTPD: Added support for managing host keys and host certificates directly from the WebAdmin UI.
- Google Cloud Storage backend: Added support for configuring the Universe Domain to allow connections to custom Google Cloud environments (e.g., Google Distributed Cloud or Sovereign Clouds).
- S3 and Azure Blob backends: improved memory usage efficiency for small file transfers. Similar optimizations were already in place for the Google Cloud Storage backend.
- Cloud Backends: Improved performance and memory efficiency of the internal cache, resulting in noticeable gains under heavy load.
- Background tasks: Optimized query logic, delivering significant performance improvements in large-scale installations.
- Event Manager: Added `stringContains` and `slicesContains` helper functions.
- Keyboard Interactive Hook: Added support for returning a custom error message and terminating the connection.
- Shares: Added `base_url` configuration option to define the external base URL used when generating public links. This setting overrides the default browser-based detection and is useful when SFTPGo is deployed behind reverse proxies, load balancers, or in NAT environments.

### Bug fixes

- SFTPD: Fixed the `SSH_FXP_REALPATH` response for the root path of virtual folders.
- FTPD: fixed an issue that could cause newly connected, unauthenticated clients to be disconnected prematurely.
- Fixed an issue where JSON dumps containing command actions failed to load correctly at startup when loaded as initial data.

### Security fix

- Fixed a potential path traversal and permission bypass involving specially crafted paths. CVE-2026-30914.

### Backward incompatible changes

- Google Cloud Storage: Starting from this version, GCS backends require standard Service Account JSON keys (`"type": "service_account"`) by default. Other credential types (`authorized_user`, `external_account`, `impersonated_service_account`) are no longer supported by default. This change improves security (mitigating risks such as SSRF) and stability by preventing the use of temporary or externally managed credentials. Users relying on non-standard JSON credentials must migrate to a Service Account key or switch to [Application Default Credentials](https://docs.cloud.google.com/docs/authentication/application-default-credentials). The previous behavior can be restored by setting the `SFTPGO_HOOK__GCS_TRUSTED_JSON_CREDENTIALS` environment variable to `1`.
- API Keys: Expiration dates can no longer be extended once set; they can only be shortened.
- Unified path handling: Prior to this release, the backslash character (`\`) was treated differently depending on the host operating system: on Linux, it was considered a standard character within a file or directory name, while on Windows, it acted as a path separator. We have now unified path handling across all platforms. Moving forward, both forward slashes (`/`) and backslashes (`\`) are strictly evaluated as path separators, independently of the underlying OS.

## Update January 20, 2026 - v2.7.20260120

### New features

- Hooks: Added the ability to automatically create virtual folders from the pre-login and pre-auth hooks by setting the `SFTPGO_HOOK__AUTO_FOLDERS` environment variable to `1`.
- Pre-login hook: Added support for returning a different username than the one used during login.
- EventManager: Added the `{{.Shares}}` lazy placeholder to retrieve the shares associated with the path on which the filesystem action was executed.
- REST API: Introduced the ability to change the log level dynamically without restarting the service.

### Bug fixes

- Data Provider: Fixed lock handling issues during migrations that could affect MySQL when migrations are executed concurrently by multiple instances.
- Shares: Fixed a bug introduced in `v2.7.20260110` that prevented uploads to write-only shares from the WebClient.

## Update January 10, 2026 - v2.7.20260110

### New features

- EventManager: Added `IMAP` action allowing to automatically retrieve email attachments from IMAP mailboxes and makes them available in SFTPGo. Attachments can be placed in a user’s home directory or mapped into a virtual folder, enabling seamless ingestion of files delivered via email.
- EventManager: Removed hard-coded subjects for emails generated from filesystem templates (outside EventManager) and made them configurable via [environment variables](env-vars.md#additional-environment-variables).
- EventManager: Added an `ICAP` action to enable integration with ICAP servers for antivirus scanning and DLP checks as part of SFTPGo rules, with automatic handling based on scan results (block, adapt, quarantine, or trigger notifications).
- EventManager: Added `ShareExpiration` action to automate [share lifecycle management](tutorials/shares.md#automating-share-lifecycle-management). It enables expiration based on inactivity thresholds or token exhaustion, with support for pre-expiration warnings (`AdvanceNotice`) and soft deletes (`GracePeriod`). The action handles group shares by notifying all members during the warning phase while restricting the deletion event to the share owner.
- SFTPD: Added the ability to [hide dot directory entries](env-vars.md#additional-environment-variables).
- Shares: Added support for associating groups with shares. This allows permissions to read, update, and delete shares to be granted to members of the same group, facilitating team [collaboration and delegation](./tutorials/shares.md#delegating-share-management).
- OIDC: Added support for configuring the claim values used to map an OIDC-authenticated user to an SFTPGo admin. Previously, only the fixed value `admin` was accepted and could not be customized.
- OIDC: Added support for `max_age` and `prompt` parameters to enforce re-authentication based on session age and control the login/consent interaction.
- WebAdmin UI / REST API: Added support for configuring [request size limits](env-vars.md#additional-environment-variables). Previously, the maximum size for HTTP request payloads was fixed at 1 MB.
- WebClient: Added support for cloning shares.
- WebAdmin / WebClient: Added Chinese and Spanish translations.
- Web UI: Added a new option in the Branding section that allows hiding the WebAdmin or WebClient link from the login page.
- Users: Added support for multiple custom placeholders (up to 10), which can be referenced in group configurations as `%custom1%`, `%custom2%`, and so on.

### Bug fixes

- EventManager: optimized filtering by pushing queries down to the database when no wildcards are used, also resolving certain incorrect results for inverse matches.
- PreLogin hook: Previously, partial user objects were accepted, which could lead to inconsistent updates (e.g., merging instead of replacing per-directory permissions). Now, the hook requires either a complete user object or an empty response if no modifications are needed.

### Security fix

- Fixed placeholder sanitization in group home directories and key prefixes. CVE-2026-30915.

### Backward incompatible changes

- Enforced stricter validation for usernames and object names: Control characters, `/`, and `\` are now rejected. These characters are included, URL-encoded, in request paths (e.g., `/web/admin/user/<username>`) and can be misinterpreted by some older proxy servers, potentially causing request routing issues.
- OAuth2: PKCE is now enabled by default to improve security. If you are connecting to a legacy OAuth2/OIDC endpoint, you can disable PKCE in your [configuration](./config-file.md#http-server).
- OIDC: Matching of role claim values used to map an OIDC-authenticated user to an SFTPGo admin is now case-insensitive.
- Public Shares API: The file upload endpoint has been updated from `POST /api/v2/shares/{id}/{filePath}` to `POST /api/v2/shares/{id}/files/{filePath}`. Previously, the file path was required to be a single, fully URL-encoded segment (e.g. `dir%2Ffile.txt`). The new endpoint supports standard slash-delimited paths (e.g. `dir/file.txt`). This change resolves 404 errors that could occur in edge cases with the previous URL structure.

## Update November 7, 2025 - v2.7.20251107

### New features

- New storage backend: Added FTP as a storage backend. This allows using an external FTP server for storage, as well as integrating FTP virtual folders with the EventManager to push or pull files over FTP/S.
- REST API: Added the `/api/v2/saas/usage` endpoint to retrieve storage and bandwidth usage for SFTPGo SaaS deployments.
- Added static password validation rules to enforce minimum length, uppercase, lowercase, digit, and special character requirements.
While this feature was introduced following multiple requests, we still recommend using the password strength setting instead, as it evaluates the overall cryptographic strength of passwords and provides stronger security.
- SSH: When the `Enforce secure algorithms` setting is enabled, public key signature algorithms are now also validated.
- Auth plugin: Import email addresses from Active Directory when creating SFTPGo users.
- EventManager: Added a new action for extracting ZIP archives.
- EventManager: Added `fromMillis` helper function to easily convert Unix timestamp expressed in milliseconds into time objects.
- EventManager: Scheduling rules now support minute-level precision.
- SFTPD: Added support for [OpenPubkey SSH](https://github.com/openpubkey/opkssh){:target="_blank"}, enabling tighter integration between OpenID Connect and SFTP.
- JWT: Replaced [lestrrat-go/jwx](https://github.com/lestrrat-go/jwx){:target="_blank"} with a lightweight wrapper around [go-jose](https://github.com/go-jose/go-jose){:target="_blank"}. Implementing our own wrapper simplifies the codebase and improves maintainability. Moreover, go-jose depends only on the standard library, resulting in a leaner dependency that still meets all our requirements.

### Bug fixes

- Enforced password validation rules also when applied through a group.
- Upload Resume: Improved and finalized the fix introduced in `v2.7.20251009`.
- Fixed an issue where `X-Forwarded-For` headers containing a port number were not handled correctly.
- IP addresses in the trusted list were never blocked, but they were still counted toward auto-blocking, which caused confusion.

## Update October 9, 2025 - v2.7.20251009

### New features

- EventManager: Introduced support for split retention report notifications, enabling separate email notifications to be sent to individual users.
- EventManager: Added the `{{.ExtName}}` placeholder, representing the email address used for authenticating public shares with email-based access.
- Status page: Added display of license information.
- WebClient: External users can now explicitly create directories in public shares.
- WebClient: Added the ability to modify the name of a public share and display it to external users.
- WebUI: Updated to Bootstrap 5.3.8, Axios 1.12.2, and the latest versions of other dependencies.
- WebUI: Added support for cloning groups, rules and actions.
- FTPD: Added TLS version and cipher suite information to the login log.
- Windows: Added support for installing the service under a custom user account via command line.

### Bug fixes

- Upload Resume: Fixed edge cases related to resuming uploads with cloud storage backends.
- WebAdmin UI: preserve condition pattern order to ensure correct filter evaluation, especially with inverse matches.
- Cloud Storage backends: Fixed a rare bug that could cause directory listings to fail under specific conditions.

### Backward incompatible changes

- Removed `rsync` support. In the previous versions, rsync was executed as an external command, which means we have no insight into or control over what it actually does. From a security perspective, this is far from ideal. To be clear, there's nothing inherently wrong with `rsync` itself. However, if we were to support it properly within SFTPGo, we would need to implement the low-level protocol internally rather than relying on launching an external process. This would ensure it works seamlessly with any storage backend, just as SFTP does, for example. We recommend using one of the many alternatives that rely on the SFTP protocol, such as `rclone`.
- Windows: To clearly differentiate the Enterprise edition from the open source version, the installer now uses a distinct GUID and installs SFTPGo into the folder named "SFTPGo Enterprise".

### Documentation

- Added a tutorial on [public shares](./tutorials/shares.md).
- Expanded the Event Manager tutorial with a [PGP](./tutorials/eventmanager-pgp.md) usage example.

## Update August 31, 2025 - v2.7.20250831

### New features

- Added support for license keys and made the Enterprise Edition generally available.
- SFTP storage backend: Added support for SOCKS proxy versions v4 and v4a.
- Preserve sort order for related folders and groups, improving compatibility and predictability when used with the Terraform provider.
- Cluster mode: ensures near real-time propagation of event rule updates, IP lists, and license changes across all cluster nodes.
- Password hashing: Introduced general support for importing passwords using the `yescrypt` format.
- EventManager: Added "Metadata Check" action to verify the presence or value of a metadata key in cloud storage backends, with optional retries and timeout.

### Bug fixes

- SMTP: Fixed OAuth2 authentication issue when using the Microsoft provider with a tenant ID.
- Fixed a rare edge case where transfers could remain stalled due to unresponsive storage backends.
- WebDAV: Fixed an issue where some scanner devices failed to process PROPFIND responses returned for GET requests on collection resources.
- Preserved the initial sort order of folders and groups to improve compatibility and ensure predictable behavior when used with Terraform.

### Backward incompatible changes

- Removed Git support. Hosting Git repositories over SSH falls outside the intended scope of a file transfer solution, and the use of external commands introduces unnecessary security risks by increasing the attack surface.

## Update July 26, 2025 - v2.7.20250726

### New features

- EventManager: Completely rewritten the action placeholder system to significantly improve flexibility. Please refer to the [migration guide](migration.md) for details and migration steps. Note that this change may break compatibility in some cases. Don’t hesitate to contact us if you need assistance migrating your actions.
- EventManager: Added an optional archive folder to the data retention action, enabling files to be moved to the specified virtual folder instead of being deleted.
- Users: Added support for a custom placeholder value set in user configurations. It can be referenced as `%custom1%` within group configurations.
- User Templates: Added support for password change requirement option.

### Bug fixes

- PGP: Fixed support for keys without flags, making behavior consistent with the `gpg` CLI tool.
- OIDC: Use the global HTTP client to ensure consistent settings, such as timeouts and TLS.
- WOPI: Use the global HTTP client to ensure consistent settings, such as timeouts and TLS. Removed `skip_tls_verify`.
- EventManager: Updated user inactivity calculation to also consider the `updated at` timestamp.
- Active Sessions: Fixed a rare edge case that could cause incorrect calculation of the number of currently active sessions.
- WebClient: Improved layout and responsiveness on mobile browsers.
- Memorypipe: Fixed edge cases that could occur during retries of multipart requests.

### Backward incompatible changes

- EventManager: Removed the Data Retention API. You can achieve equivalent functionality using the Data Retention Check action.
- EventManager: Removed several placeholders. Refer to the [migration guide](migration.md) to update your existing actions accordingly.
- SFTPFs: Removed the root directory existence check. If a root path is configured, it is now assumed to exist, avoiding redundant and time-consuming validations. This change improves login times in environments with multiple virtual folders backed by SFTP, particularly when one or more of the backend SFTP storage endpoints are unreachable or experiencing timeouts.

## Update July 2, 2025 - v2.7.20250702

### New features

- EventManager: Added support for folder integration to simplify pushing files to external SFTP servers and other storage backends, either after an upload or on a scheduled basis. Tutorials have been updated [with examples](tutorials/eventmanager-folders.md) demonstrating folders integration.
- WebAdmin: Added the ability to disable password authentication for administrators. This is useful if you want to enforce API key–only access or restrict login to OpenID Connect.
- Audit Log: Log entries for configuration changes now include details about the specific modified section.
- Terraform Provider: Updated to fully support SFTPGo Enterprise.
- Extend metadata propagation in cloud storage to include `delete`, `rename`, and `copy` events. Support for these features depends on the storage backend: for example, metadata may not always be available with S3, whereas it is consistently supported with Google Cloud Storage and Azure Blob Storage.
- WebUI: Added autofocus to the login and 2FA input fields, improving the user experience by automatically focusing on the first field when the page loads.
- REST API documentation for SFTPGo Enterprise is now hosted on the [sftpgo.com](https://sftpgo.com/rest-api){:target="_blank"} domain.

## Update June 7, 2025 - v2.7.20250607

### New features

- Google Cloud Storage: Added support for Hierarchical Namespace enabled buckets.
- OIDC: The label "Sign in with OpenID" on the login screen is now configurable.
- OIDC: added support for discovery when the issuer URL and discovery URL differ (e.g. Azure B2C). This behavior is disabled by default and can be enabled by setting the environment variable `SFTPGO_HTTPD__BINDINGS__0__OIDC__INSECURE_ISSUER_URL` to `1`.
- FTPD: implemented the STRU command to improve compatibility with legacy FTP clients.
- Shares: added audit logging for legal agreement acceptance.

### Bug fixes

- Storage sync plugin: fixed handling of unexpectedly interrupted sync operations to ensure proper recovery and consistency.

## Update May 18, 2025 - v2.7.20250518

### New features

- Cloud backends: Added support for the following new environment variables: `SFTPGO_HOOK__GCS_CHECK_PARENT_DIR`, `SFTPGO_HOOK__S3_CHECK_PARENT_DIR`, `SFTPGO_HOOK__AZBLOB_CHECK_PARENT_DIR`. When set to `1`, these variables prevent uploads to non-existent directories. Cloud backends do not use real directories, so uploading to non-existent paths typically works without errors.
- SFTPD: Added support for restricting insecure algorithms on a per-user basis. In the Advanced Settings section, you can enable the "Enforce secure algorithms" checkbox for each user or group. This will disable weak host keys, key exchange, MAC, and cipher algorithms that are enabled system-wide.
- WOPI: Added support for skipping specific file extensions.
- Plugins: Added storage sync.
- REST API can be disabled on a per-user basis.

### Bug fixes

- WebClient: Fixed an issue with recursive folder deletion.
- WebClient: Leading and trailing spaces are now allowed in user passwords, improving compatibility with certain external identity providers.
- WebClient: Fixed issue with incorrect behavior in multi-page selection.

## Update April 22, 2025 - v2.7.20250422

### New features

- EventManager: All placeholder names must now include a leading dot, changing from `{{PlaceholderName}}` to `{{.PlaceholderName}}`. For example, use `{{.VirtualPath}}` instead of `{{VirtualPath}}`. Existing actions are automatically migrated to the new format. When adding new actions, make sure to use the new format. This change is a preparation to improve flexibility in the future, allowing for features like conditions, loops, and other advanced options.
- EventManager: Added support for PGP encryption and decryption.
- HTTPD: Allowed to configure the `Referrer-Policy` header.
- SFTPD: Added support for Post-Quantum Traditional Hybrid Key Exchange through the newly added algorithm `mlkem768x25519-sha256`.

Some notes about `mlkem768x25519-sha256`:

- Enabled by default: no configuration changes are needed. If both client and server support it, the hybrid KEX will be used automatically.
- Fully interoperable: clients that don't yet support post-quantum algorithms like `mlkem768x25519-sha256` will continue to work as expected using standard key exchange algorithms—ensuring a smooth and backward-compatible experience.

### Bug fixes

- Respect the configured naming rules also for OpenID Connect authentication.

## Update April 5, 2025 - v2.7.20250405

### New features

- Azure Blob Storage Backend: General performance enhancements, including all the optimizations previously applied to S3 and Google Cloud storage.
- AES CTR Ciphers: Enhanced performance by 2x, with CTR mode now offering performance comparable to GCM mode.
- A legal agreement can be displayed before granting access to the share by external users. This feature can be enabled by setting the environment variable `SFTPGO_HOOK__HAS_SHARE_LEGAL_AGREEMENT` to `1`. The default legal agreement is located in the SFTPGo templates directory (`templates/webclient/sharelegal.html`). To customize it, you can specify a new template by setting the `SFTPGO_HOOK_SHARE_LEGAL_TMPL_PATH` environment variable to the path of the desired template. We recommend using the default template as a starting point.
- Web UIs: Added support for French and German localizations.

Here is an example configuration to enable all the supported localizations using environment variables.

```shell
SFTPGO_HTTPD__BINDINGS__0__LANGUAGES=en,it,de,fr
```

### Bug fixes

- Fixed issue with loading user data after an update from the pre-login hook. Previously, related fields such as virtual folders and groups were not refreshed, causing inconsistent user data.
- Enabled login via OpenID Connect even when password-based login is disabled.
- Fixed a security issue in an SFTPGo dependency; more details, including a CVE link, will be added once the security issue is made public.

## Update March 27, 2025 - v2.7.20250327

### New features

- SFTP storage backend: Added SOCKS5 proxy support.
- WebClient: Integrated the WOPI protocol, enabling the opening and editing of Office files through a compatible document server (e.g. Collabora Online, OnlyOffice)

Here is an example configuration for the WOPI integration using environment variables.

```shell
SFTPGO_HTTPD__BINDINGS__0__WOPI__SERVER_URL="http://192.168.1.136"
SFTPGO_HTTPD__BINDINGS__0__WOPI__CALLBACK_URL="http://192.168.1.148:8080"
SFTPGO_HTTPD__BINDINGS__0__WOPI__SKIP_TLS_VERIFY=0
SFTPGO_HTTPD__BINDINGS__0__WOPI__SKIP_PROOF_KEY_VERIFY=0
SFTPGO_HTTPD__BINDINGS__0__WOPI__ALLOWED_FROM=192.168.1.0/24,192.168.2.25
```

The required configuration settings are the `server_url` and the `callback_url`, the other settings are optional.
Additionally, you can configure the lifetime of WOPI access tokens (default is 360 minutes) by setting the environment variable `SFTPGO_HTTPD__WOPI_TOKEN_LIFETIME`. The value must be specified in minutes.

- `server_url`, defines the base URL of your document server. The URL where your document server is reachable.
- `callback_url`, defines the base URL that the document server uses to save your documents. This is the base URL where SFTPGo is reachable.
- `skip_tls_verify`, skips TLS validation. Maybe be useful if your document server uses a self-signed certificate. Enabling this setting is not recommended.
- `skip_proof_key_verify`, if your document server supports [proof keys](https://learn.microsoft.com/en-us/microsoft-365/cloud-storage-partner-program/online/scenarios/proofkeys), SFTPGo will validate them to ensure that incoming WOPI requests are originating from your document server. Disabling this setting is not recommended.
- `allowed_from`, allows to define IP addresses and ranges allowed to perform WOPI requests. This setting can be configured for servers not supporting proof keys or in addition to them.

### Bug fixes

- Public shares: Show the configured disclaimer on the login page.
- SMTP: Added support for servers that do not advertise the SMTP AUTH extension.

## Update March 08, 2025 - v2.7.20250308

### New features

- EventManager: date/time placeholders like `{{DateTime}}`, `{{Year}}`, `{{Month}}`, `{{Day}}`, `{{Hour}}`, `{{Minute}}` can now be used in actions started on schedule.
- EventManager: the `{{Email}}` placeholder now expands to multiple email addresses. For example, you can define multiple email addresses for a user - before this change, only the primary email address was taken into account.
- WebClient: External users can now authenticate to shared files and folders using their email address; each public share can be restricted to one or more specified email addresses, and when email authentication is enabled, users must enter their email and will receive a one-time authentication code via email to complete authentication.

### Bug fixes

- WebUI: the column visibility feature now hides the correct columns even after reordering table columns.
- WebUI: fixed context menu activation in user lists and other tables. In some edge cases the menu was not displayed because it was activated too early in the page rendering.
- WebUI: hidden some advanced settings like part size and concurrency for cloud storage backends. They are confusing for users and the values are closely related to instance resources, so now they are adjusted automatically.

## Other additions compared to the Open Source edition

- Performance improvements for the cloud storage backends, especially when uploading numerous small files.
- Downloads and uploads to cloud storage backends can be fully handled in memory by setting the environment variable `SFTPGO_HOOK__MEMORY_PIPES__ENABLED` to `1`. No unlinked files will be created, and therefore no local storage space will be used.
- Added new configuration parameters to specify the maximum number of transfers (downloads and uploads) allowed, both in total and per host. The Open Source edition only allows limiting total and per-host connections, not transfers. By default the maximum number of transfers allowed per host is 20.
- Added Trusted List. IP addresses or networks added to this list are always trusted, exempt from blocking by the Defender and Geo-IP filtering, and will never be subject to rate limiting.
