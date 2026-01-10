# Release Notes

This page provides a concise overview of the new features, improvements and bug fixes introduced in each SFTPGo Enterprise release.
We encourage you to check back regularly to stay up to date with the latest changes and to make the most of all enhancements.

## Compatibility notes

Upgrading to the Enterprise edition of SFTPGo is supported starting from Open Source release 2.6.x.

If you're migrating from an open-source installation, please follow the guide here: [**Migration from Open-Source 2.6.x to Enterprise**](tutorials/migrating.md)

## Update January 10, 2026 - v2.7.20260110

### New features

- EventManager: Added `IMAP` action allowing to automatically retrieves email attachments from IMAP mailboxes and makes them available in SFTPGo. Attachments can be placed in a user’s home directory or mapped into a virtual folder, enabling seamless ingestion of files delivered via email.
- EventManager: Removed hard-coded subjects for emails generated from filesystem templates (outside EventManager) and made them configurable via [environment variables](env-vars.md#additional-environment-variables).
- EventManager: Added an `ICAP` action to enable integration with ICAP servers for antivirus scanning and DLP checks as part of SFTPGo rules, with automatic handling based on scan results (block, adapt, quarantine, or trigger notifications).
- EventManager: Added `ShareExpiration` action to automate [share lifecycle management](tutorials/shares.md#automating-share-lifecycle-management). It enables expiration based on inactivity thresholds or token exhaustion, with support for pre-expiration warnings (`AdvanceNotice`) and soft deletes (`GracePeriod`). The action handles group shares by notifying all members during the warning phase while restricting the deletion event to the share owner.
- SFTPD: Allow to [hide dot directory entries](env-vars.md#additional-environment-variables).
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
- Expanded the Event Manager tutorial with a [PGP](./tutorials/eventmanager.md#pgp-compatible-encryption-and-decryption) usage example.

## Update August 31, 2025 - v2.7.20250831

### New features

- Added support for license keys and made the Enterprise Edition generally available.
- SFTP storage backend: Added support for SOCKS proxy versions v4 and v4a.
- Preserve sort order for related folders and groups, improving compatibility and predictability when used with the Terraform provider.
- Cluster mode: ensures near real-time propagation of event rule updates, IP lists, and license changes across all cluster nodes.
- Password hashing: Introduced general support for importing password using the `yescrypt` format.
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
