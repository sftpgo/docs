---
description: "SFTPGo features: secure file transfer, workflow automation, encryption, audit logging, SSO, cloud storage, HA clustering, Kubernetes, and Terraform IaC support."
---

# Features

SFTPGo provides enterprise-grade managed file transfer capabilities: multi-protocol secure access, event-driven workflow automation, data-at-rest encryption, centralized audit logging, granular access controls, and integration with cloud storage and identity providers.

## Protocols

SFTPGo provides access to files through multiple protocols. Users, permissions, storage backends, and event rules apply consistently across all of them.

- [SFTP/SCP](ssh.md) over SSH, with post-quantum hybrid key exchange (`mlkem768x25519-sha256`), configurable ciphers, KEX, MACs, and host key algorithms.
- [FTP/FTPS](ftp.md) with explicit and implicit TLS, mutual TLS authentication, and session reuse.
- [WebDAV](webdav.md) over HTTP/HTTPS with lock support.
- Built-in [WebClient](web-interfaces.md#webclient) over HTTPS for browser-based file management and sharing.
- [TUS](https://tus.io/){:target="_blank"} resumable upload protocol for chunked, resumable uploads via the WebClient and the REST API. Improves reliability in unstable network conditions and compatibility with proxy/CDN environments such as Cloudflare.
- Support for [HAProxy PROXY protocol](config-file.md) to preserve client IP addresses behind load balancers and proxies.

## Storage backends

SFTPGo abstracts storage so that users and administrators work with a unified interface regardless of where files physically reside. Multiple backends can be combined within the same installation — even for the same user via virtual folders.

- [Local filesystem](localfs.md) with per-user home directory isolation.
- [Data At Rest Encryption](dare.md) (CryptFs) — transparent AES-256-GCM / ChaCha20-Poly1305 encryption on the local filesystem.
- [S3-compatible object storage](s3.md) (AWS S3, Wasabi, Backblaze B2, etc.) with multipart transfers, SSE-C encryption, and IAM role support.
- [Google Cloud Storage](google-cloud-storage.md) with Hierarchical Namespace (HNS) support.
- [Azure Blob Storage](azure-blob-storage.md) with shared key, SAS, and default Azure credential authentication.
- [Remote SFTP servers](sftpfs.md) as storage, with optional buffering, SOCKS proxy support, and configurable concurrency.
- [Remote FTP servers](ftpfs.md) as storage, with explicit and implicit TLS support.
- [Custom HTTP backends](httpfs.md) via a REST API contract, for integration with arbitrary storage systems.
- [Virtual folders](virtual-folders.md) to map directories from any supported backend into a user's namespace. Virtual folders can be private or shared, each with independent quota.

## Authentication and security

SFTPGo supports a wide range of authentication methods and security controls, from standard password/key authentication to enterprise SSO and automatic threat mitigation.

- Password, public key, and certificate authentication for SSH. Password and mutual TLS for FTP and WebDAV.
- Multi-factor authentication (TOTP) compatible with Microsoft Authenticator, Google Authenticator, Authy, and similar apps.
- Multi-step authentication — combine methods per user (e.g., public key + password).
- [OpenID Connect](oidc.md) Single Sign-On, supporting Microsoft Entra ID, Google, Amazon Cognito, Auth0, Okta, OneLogin, JumpCloud, Ping Identity, Keycloak, and others. Supports [PKCE without client secret](oidc.md#pkce-without-client-secret) for public OAuth2 clients.
- LDAP/Active Directory integration for user authentication and group mapping.
- Custom authentication via [external programs or HTTP hooks](external-auth.md).
- Dynamic user creation or modification before login via [pre-login hooks](dynamic-user-mod.md).
- Geo-IP filtering to allow or deny access by country.
- Per-user and global IP allow/deny lists with a trusted list that bypasses the defender and rate limiting.
- Automatic brute-force protection via the built-in [defender](defender.md) with configurable scoring and ban policies.
- Per-protocol [rate limiting](rate-limiting.md) with per-IP and global limiters.
- Strict Content Security Policy support — no `unsafe-eval` or `unsafe-inline` required.
- Configurable SSH algorithms (ciphers, KEX, MACs, host keys) with per-user enforcement of secure algorithms.
- Post-quantum hybrid key exchange for SSH (`mlkem768x25519-sha256`) and TLS 1.3 (X25519MLKEM768), protecting file transfers against future quantum computing threats while remaining fully backward compatible.
- Let's Encrypt TLS certificates via built-in ACME protocol for HTTPS, FTPS, and WebDAV. TLS certificates can also be uploaded directly from the WebAdmin UI.
- Access time restrictions per user.

## User and access management

SFTPGo provides granular control over what each user can do, where they can store files, and how much they can transfer.

- Users stored in SQLite, MySQL, PostgreSQL, CockroachDB, Bolt, or in-memory, each restricted to their home directory or bucket prefix.
- Granular per-user and per-directory permissions (list, download, upload, overwrite, delete, create directories, rename, create symlinks, chmod).
- [Groups](groups.md) to define settings once and apply them to multiple users (primary, secondary, and membership groups with inheritance rules).
- [Roles](roles.md) for delegated administration — restricted administrators who can only manage users with the same role.
- Disk quota management per user and per virtual folder (total size and/or file count).
- Bandwidth throttling with separate upload/download limits, configurable per client IP.
- Transfer quota with combined or separate upload/download limits, resettable via REST API or Event Manager.
- Automatic deactivation or deletion of inactive users.
- Automatic termination of idle connections.

## Sharing

Users can securely share files and folders with external parties via the WebClient, with fine-grained controls over access and lifecycle.

- Public file and folder shares via HTTP/S links, with configurable limits (download/upload count, expiration date, source IP restrictions).
- Password and email-based authentication for public shares.
- Group-based share delegation and [governance policies](tutorials/shares.md).
- [Denied share paths](tutorials/shares.md#restricting-shareable-paths) — administrators can block specific virtual paths from being shared, per-user or inherited from groups.
- [Denied share scopes](tutorials/shares.md#restricting-shareable-paths) — restrict which share types (read, write, read/write) are available. When all scopes are denied, sharing is completely disabled.
- Share lifecycle management with inactivity thresholds, advance notice, and grace periods.
- Share cloning for quickly creating variants of existing shares.

## Automation

The [Event Manager](eventmanager.md) is SFTPGo's automation engine — it lets administrators define rules that react to events and execute actions automatically, with full [template](placeholders.md) support.

- Triggers: file operations (`upload`, `download`, `delete`, `rename`, `copy`, `mkdir`, `rmdir`), provider changes (`add`, `update`, `delete`), schedules, IP blocks, certificate renewals, on-demand, and Identity Provider logins.
- Filesystem actions: [copy](filesystem-actions.md#copy) (with source disposition, glob patterns, per-file retries, continue-on-error), [rename, delete](filesystem-actions.md#delete) (including directory contents), [compress](filesystem-actions.md#compress) (ZIP), [extract](filesystem-actions.md#extract) (ZIP), create directories, path existence checks.
- [Virtual folder integration](eventmanager.md#virtual-folders) for cross-backend operations (e.g., copy uploads to a different S3 bucket, push files to an external SFTP server, archive to a remote location).
- [Execute Before File Publish](execute-before-file-publish.md): staged upload actions that scan or validate files before they become visible (e.g., antivirus via ICAP, content validation).
- [ICAP integration](filesystem-actions.md#icap) for antivirus scanning and DLP checks on all storage backends, with quarantine support via virtual folders.
- [IMAP integration](filesystem-actions.md#imap) for automatic ingestion of email attachments.
- PGP encryption and decryption with optional signing and signature verification.
- [Event reports](event-report.md) for periodic digest notifications of filesystem activity, with per-user split reports and CSV attachments.
- Data retention with per-directory policies — expired files can be automatically [deleted or archived](tutorials/eventmanager-retention.md) to a virtual folder.
- [Auto provisioning](tutorials/eventmanager-idp.md) — automatically create user accounts when they log in through an Identity Provider.
- Notifications via email, HTTP webhooks, and command execution, with customizable templates.
- Path filter conditions can match on [virtual path or filesystem path](eventmanager.md#path-filters), enabling filtering by physical storage location.
- Configurable [custom commands and HTTP hooks](custom-actions.md) for file and provider operations.

See the [Event Manager tutorials](tutorials/eventmanager.md) for step-by-step guides covering common workflows.

## Web interfaces

SFTPGo includes two web interfaces, both with dark and light themes and support for six languages (English, Italian, German, French, Spanish, Chinese Simplified).

- [WebAdmin](web-interfaces.md#webadmin) for centralized management of users, groups, virtual folders, event rules, and server configuration. Includes a global [configurations page](initial-configuration.md#configuration) for managing SMTP, OIDC, LDAP, TLS certificates, email templates, and language settings directly from the UI.
- [WebClient](web-interfaces.md#webclient) with browser-based file management, credential management, two-factor authentication setup, and file sharing.
- WOPI protocol integration for in-browser document editing and real-time collaboration (e.g., with Collabora Online).
- Branding: custom logo, name, favicon, and login page disclaimer for both interfaces.

## High availability and deployment

SFTPGo is designed to scale from a single instance to large distributed deployments.

- Multi-node clustering with near real-time propagation of configuration changes across all nodes.
- Native Kubernetes support with an official [Helm chart](installation.md).
- Available on the [AWS Marketplace](tutorials/sftpgo-aws.md), [Azure Marketplace](installation.md#azure), and [Google Cloud Marketplace](tutorials/sftpgo-google-cloud.md).
- Runs on Linux, Windows, macOS, and FreeBSD — from embedded devices to cloud infrastructure.
- [Docker images](docker.md) for containerized deployments.

## Administration and operations

- [REST API](rest-api.md) for full system management and end-user file operations, with OpenAPI documentation. Supports JWT and [API key](rest-api.md#api-key-authentication) authentication. API keys can be created and managed from the WebAdmin UI.
- [Terraform provider](https://registry.terraform.io/providers/drakkan/sftpgo/latest){:target="_blank"} for Infrastructure as Code — manage users, groups, folders, and event rules declaratively.
- Real-time monitoring of active connections.
- [Audit logs](plugins/audit-logs.md) with structured JSON output for file transfers, logins, SSH commands, and HTTP requests. Searchable from the WebAdmin UI.
- [Prometheus metrics](metrics.md) for monitoring and alerting, including per-user transfer metrics.
- TLS certificate management directly from the WebAdmin UI, as an alternative to Let's Encrypt/ACME.
- Built-in [profiler](profiling.md) for performance analysis.
- [Portable mode](cli.md#portable-mode) for quick, on-demand file sharing from a single directory.

## Integration and extensibility

- [Plugin system](plugins.md) for audit logs, LDAP authentication, GeoIP filtering, cloud KMS, and pub/sub event forwarding.
- Migration tools for importing users from [Linux system accounts](https://github.com/drakkan/sftpgo/tree/main/examples/convertusers){:target="_blank"}.
