---
description: "Documentation map for SFTPGo — find the right page for each task: install, storage, users, groups, Event Manager, OIDC, shares, security, automation."
---

# Documentation Map

A task-oriented index: find the page that answers your current question, without scrolling through the full navigation.

If you are looking for the definition of a term, see the [Glossary](glossary.md).

## Getting started

| I want to... | Go to |
| -------------- | ------- |
| Understand what SFTPGo is and which features it has | [Features](features.md) |
| Install SFTPGo on Linux, Windows, macOS, Docker, or Kubernetes | [Installation](installation.md), [Docker](docker.md) |
| Do the first-time setup (admin, first user, test login) | [Getting Started](initial-configuration.md) |
| Use example configurations for common deployments | [Configuration Examples](configuration-examples.md) |

## Users, groups, roles

| I want to... | Go to |
| -------------- | ------- |
| Create a user (any backend) | [Getting Started](initial-configuration.md), [REST API](rest-api.md) |
| Restrict what a user can see and do (permissions, file patterns, isolation) | [Access Control](access-control.md) |
| Expose only selected directories to a user | [Access Control](access-control.md), [Virtual Folders](virtual-folders.md) |
| Use groups to centralize settings | [Groups](groups.md), [Groups tutorial](tutorials/groups-example.md) |
| Understand primary vs secondary vs membership groups | [Groups](groups.md) |
| Delegate admin duties to scoped administrators | [Roles](roles.md), [Admin Permissions](admin-permissions.md) |
| Grant specific admin capabilities (helpdesk, catalog steward, provisioning bot) | [Admin Permissions](admin-permissions.md) |
| Let users self-change their password | [WebClient](web-interfaces.md#webclient) |

## Storage backends

| I want to... | Go to |
| -------------- | ------- |
| Use the local filesystem | [Local filesystem](localfs.md) |
| Use encrypted local storage | [Data At Rest Encryption](dare.md) |
| Use Amazon S3 or S3-compatible storage (Wasabi, MinIO, etc.) | [S3](s3.md) |
| Use Google Cloud Storage | [GCS](google-cloud-storage.md) |
| Use Azure Blob Storage | [Azure Blob](azure-blob-storage.md) |
| Chain SFTPGo on top of another SFTP server | [SFTP backend](sftpfs.md) |
| Use an FTP server as a backend | [FTP backend](ftpfs.md) |
| Use a custom HTTP API as storage | [HTTPFs](httpfs.md) |
| Mount different backends at different paths | [Virtual Folders](virtual-folders.md) |

## Protocols

| I want to... | Go to |
| -------------- | ------- |
| Configure SFTP/SCP (SSH) | [SSH](ssh.md) |
| Configure FTP / FTPS | [FTP](ftp.md) |
| Configure WebDAV | [WebDAV](webdav.md) |
| Enable TUS resumable uploads | [`upload_chunk_size`](config-file.md), [Running behind Cloudflare](tutorials/cloudflare.md) |

## Authentication

| I want to... | Go to |
| -------------- | ------- |
| Accept SSH public keys | [SSH](ssh.md) |
| Integrate an OIDC Identity Provider (Entra ID, Okta, Google, Auth0, Keycloak…) | [OIDC](oidc.md), [Auto-provision users via IdP](tutorials/eventmanager-idp.md) |
| Integrate LDAP / Active Directory | [LDAP plugin](plugins/ldap-auth.md) |
| Enable multi-factor authentication | [Two-factor authentication](tutorials/two-factor-authentication.md) |
| Use API keys for automation | [REST API](rest-api.md) |
| Replace the built-in login with a custom service | [External auth hook](external-auth.md), [Keyboard-interactive](keyboard-interactive.md) |
| Modify user records at login time | [Pre-login hook](dynamic-user-mod.md) |

## Sharing

| I want to... | Go to |
| -------------- | ------- |
| Let users create public / password-protected links | [Shares tutorial](tutorials/shares.md) |
| Require email authentication on shares | [Shares tutorial](tutorials/shares.md) |

## Automation (Event Manager)

| I want to... | Go to |
| -------------- | ------- |
| Understand the Event Manager overview | [Event Manager](eventmanager.md), [Event Manager tutorial](tutorials/eventmanager.md) |
| Write templates with placeholders and helper functions | [Placeholders & Templates](placeholders.md) |
| Send email or webhook notifications on upload | [Upload notifications & webhooks](tutorials/eventmanager-notifications.md) |
| Auto-provision users via OIDC | [Auto-provision via IdP](tutorials/eventmanager-idp.md) |
| Schedule retention / cleanup | [Data retention](tutorials/eventmanager-retention.md) |
| Copy or archive files across backends | [Copy & archive workflows](tutorials/eventmanager-copy.md) |
| Scan uploads with antivirus (ICAP) | [Antivirus (ICAP)](tutorials/eventmanager-icap.md) |
| Encrypt/decrypt with PGP | [PGP encryption & decryption](tutorials/eventmanager-pgp.md) |
| Build auto folder structures | [Automatic folder structure](tutorials/eventmanager-auto-dirs.md) |
| Take daily backups | [Daily backups](tutorials/eventmanager-backup.md) |
| Integrate virtual folders with events | [Virtual folders integration](tutorials/eventmanager-folders.md) |
| Implement a recycle bin | [Recycle bin](tutorials/eventmanager-recycle-bin.md) |
| Run actions before a file is published to clients | [Execute before file publish](execute-before-file-publish.md) |
| See what a rule actually does in production | [Event report](event-report.md) |
| Execute filesystem actions from rules | [Filesystem actions](filesystem-actions.md) |
| Migrate from older custom action hooks | [Event Manager migration guide](migration.md) |

## Security and protection

| I want to... | Go to |
| -------------- | ------- |
| Get TLS certificates automatically | [Let's Encrypt tutorial](tutorials/lets-encrypt-certificate.md) |
| Allow access only from a curated set of IPs (default-deny) | [IP Lists](ip-lists.md) |
| Exempt trusted IPs from banning, rate limiting, and GeoIP filtering | [IP Lists](ip-lists.md) |
| Block brute-force attackers | [Defender](defender.md) |
| Rate-limit requests | [Rate limiting](rate-limiting.md) |
| Preserve client IPs behind a load balancer | [Configuration file](config-file.md) |
| Store secrets in a cloud KMS | [KMS](kms.md), [Cloud KMS providers plugin](plugins/kms-providers.md) |
| Restrict shares by IP country | [GeoIP plugin](plugins/geoip-filter.md) |

## Configuration and operation

| I want to... | Go to |
| -------------- | ------- |
| Configure via config file | [Config file](config-file.md) |
| Configure via environment variables | [Env vars](env-vars.md) |
| Use the CLI for administrative tasks | [CLI](cli.md) |
| Customize email templates | [Email templates](email-templates.md) |
| Use the WebAdmin or WebClient | [Web interfaces](web-interfaces.md) |
| Pick a data provider (SQLite, PostgreSQL, MySQL, CockroachDB) | [Data provider](data-provider.md) |
| Migrate from OSS 2.6.x to Enterprise | [Migration from OSS](tutorials/migrating.md) |

## Observability

| I want to... | Go to |
| -------------- | ------- |
| Read server logs | [Logs](logs.md) |
| Collect Prometheus metrics | [Metrics](metrics.md) |
| Store audit logs in a plugin | [Audit logs plugin](plugins/audit-logs.md) |
| Forward events to Pub/Sub or a message broker | [Pub/Sub plugin](plugins/pubsub.md) |
| Profile CPU or memory usage | [Profiling](profiling.md) |

## Integrations

| I want to... | Go to |
| -------------- | ------- |
| Deploy on AWS | [SFTPGo on AWS](tutorials/sftpgo-aws.md) |
| Deploy on Google Cloud | [SFTPGo on Google Cloud](tutorials/sftpgo-google-cloud.md) |
| Use OAuth2 for SMTP / IMAP | [OAuth2 for SMTP & IMAP](tutorials/oauth2-smtp-imap.md) |
| Extend SFTPGo with a plugin | [Plugins](plugins.md) |

## Compliance and reference

| I want to... | Go to |
| -------------- | ------- |
| Review SFTPGo's compliance posture | [Security & Compliance](compliance.md) |
| Consult the REST API contract | [REST API](rest-api.md) + `openapi/openapi.yaml` (or <https://sftpgo.com/assets/openapi.yaml>) |
| Look up a specific term | [Glossary](glossary.md) |
| See the changelog | [Changelog](changelog.md) |

## AI-assisted workflows

| I want to... | Go to |
| -------------- | ------- |
| Use Claude Code, Cursor, Copilot, ChatGPT, Gemini, or any other AI to write SFTPGo configuration, REST API payloads, WebAdmin form values, Event Action templates, or troubleshoot a deployment | [AI Assistants](ai-assistants.md) |
