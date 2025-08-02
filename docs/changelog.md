# Release Notes

This page provides a concise overview of the new features, improvements and bug fixes introduced in each SFTPGo Enterprise release.
We encourage you to check back regularly to stay up to date with the latest changes and to make the most of all enhancements.

## Compatibility notes

Upgrading to the Enterprise edition of SFTPGo is supported starting from Open Source release 2.6.x.

## Update July 26, 2025 - v2.7.20250726

### New features

- EventManager: Completely rewritten the action placeholder system to significantly improve flexibility. Please refer to the [documentation](eventmanager.md) for details and migration steps. Note that this change may break compatibility in some cases. Don’t hesitate to contact us if you need assistance migrating your actions.
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
- EventManager: Removed several placeholders. Refer to the [migration guide](eventmanager.md#migration-from-previous-versions-or-the-open-source-edition) to update your existing actions accordingly.
- SFTPFs: Removed the root directory existence check. If a root path is configured, it is now assumed to exist, avoiding redundant and time-consuming validations. This change improves login times in environments with multiple virtual folders backed by SFTP, particularly when one or more of the backend SFTP storage endpoints are unreachable or experiencing timeouts.

## Update July 2, 2025 - v2.7.20250702

### New features

- EventManager: Added support for folder integration to simplify pushing files to external SFTP servers and other storage backends, either after an upload or on a scheduled basis. Tutorials have been updated [with examples](tutorials/eventmanager.md#virtual-folders-integration) demonstrating folders integration.
- WebAdmin: Added the ability to disable password authentication for administrators. This is useful if you want to enforce API key–only access or restrict login to OpenID Connect.
- Audit Log: Log entries for configuration changes now include details about the specific modified section.
- Terraform Provider: Updated to fully support SFTPGo Enterprise.
- Extend metadata propagation in cloud storage to include `delete`, `rename`, and `copy` events. Support for these features depends on the storage backend: for example, metadata may not always be available with S3, whereas it is consistently supported with Google Cloud Storage and Azure Blob Storage.
- WebUI: Added autofocus to the login and 2FA input fields, improving the user experience by automatically focusing on the first field when the page loads.
- REST API documentation for SFTPGo Enterprise is now hosted on the [sftpgo.com](https://sftpgo.com/rest-api){:target="_blank"} domain.

## Update June 7, 2025 - v2.7.20250607

### New features

- Google Cloud Storage: Added support for Hierarchical Namespace enabled buckets.
- OIDC: UI display name is now configurable. The label "Sign in with OpenID" on the login screen is now configurable. You can change `OpenID` to something else.
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

- EventManager: All placeholder names must now include a leading dot, changing from `{{PlaceholderName}}` to `{{.PlaceholderName}}`. For example, use `{{.VirtualPath}}` instead of `{{VirtualPath}}`. Existig actions are automatically migrated to the new format. When adding new actions, make sure to use the new format. This change is a preparation to improve flexibility in the future, allowing for features like conditions, loops, and other advanced options.
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
- WebUI: hidden some advanced settings like part size and concurrency for cloud storage backends. They are confusing for users and the values ​​are closely related to instance resources, so now they are adjusted automatically.

## Other additions compared to the Open Source edition

- Performance improvements for the cloud storage backends, especially when uploading numerous small files.
- Downloads and uploads to cloud storage backends can be fully handled in memory by setting the environment variable `SFTPGO_HOOK__MEMORY_PIPES__ENABLED` to `1`. No unlinked files will be created, and therefore no local storage space will be used.
- Added new configuration parameters to specify the maximum number of transfers (downloads and uploads) allowed, both in total and per host. The Open Source edition only allows limiting total and per-host connections, not transfers. By default the maximum number of transfers allowed per host is 20.
- Added Trusted List. IP addresses or networks added to this list are always trusted, exempt from blocking by the Defender and Geo-IP filtering, and will never be subject to rate limiting.
