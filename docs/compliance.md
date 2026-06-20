---
description: "SFTPGo security capabilities for regulatory compliance: encryption, audit logging, access controls, SSO, and brute-force protection for HIPAA, GDPR, SOC 2, and DORA."
---

# Security and Compliance

SFTPGo provides a comprehensive set of security capabilities that can help organizations meet the requirements of regulatory frameworks such as HIPAA, GDPR, SOC 2, PCI DSS, and DORA. This page maps SFTPGo features to common compliance areas.

## Encryption

| Requirement | SFTPGo capability |
| ----------- | ----------------- |
| Encryption in transit | All supported protocols use encrypted transport: SFTP over SSH, FTP with explicit/implicit TLS, WebDAV over HTTPS, and the WebClient over HTTPS. Configurable cipher suites and TLS versions. Post-quantum hybrid key exchange is available for both [SSH](ssh.md) (`mlkem768x25519-sha256`) and TLS 1.3 (`X25519MLKEM768`), providing protection against future quantum computing threats while maintaining backward compatibility with existing clients. |
| Encryption at rest | [Data At Rest Encryption](dare.md) (CryptFs) provides transparent AES-256-GCM encryption on the local filesystem. Cloud backends (S3, Azure Blob, GCS) support server-side encryption. |
| Key management | Encryption keys can be managed locally or through cloud KMS providers (AWS KMS, Azure Key Vault, Google Cloud KMS, HashiCorp Vault, Oracle Key Vault). See [KMS](kms.md). |
| FIPS 140-3 | [FIPS builds](fips.md) use a FIPS-validated cryptographic module and restrict TLS, SSH, password hashing, KMS, and data-at-rest encryption to FIPS-approved algorithms. |

## Audit logging

| Requirement | SFTPGo capability |
| ----------- | ----------------- |
| Activity tracking | All file operations (upload, download, delete, rename, copy, mkdir, rmdir) and SSH commands are logged with timestamp, username, client IP, protocol, file path, and status. |
| Administrative audit trail | All changes to users, groups, admins, event rules, shares, and other configuration objects are logged with the identity of who made the change. |
| Authentication logging | Successful and failed login attempts are logged, including the authentication method and client IP. |
| Searchable audit logs | The [Audit Logs](plugins/audit-logs.md) plugin stores events in PostgreSQL, MySQL, or SQLite with a searchable UI in WebAdmin and export capabilities. |
| Log forwarding | Events can be forwarded in real time to external systems via the [Pub/Sub](plugins/pubsub.md) plugin (supports Amazon SNS/SQS, Azure Service Bus, Google Cloud Pub/Sub, RabbitMQ, NATS, Kafka). |

## Access controls

| Requirement | SFTPGo capability |
| ----------- | ----------------- |
| Authentication | Password, SSH public key, TLS client certificate, keyboard-interactive, and multi-step authentication (e.g., public key + password). |
| Multi-factor authentication | TOTP-based [two-factor authentication](tutorials/two-factor-authentication.md) compatible with standard authenticator apps. |
| Single Sign-On | [OpenID Connect](oidc.md) integration with Microsoft Entra ID, Google, Okta, Auth0, Keycloak, and others. [LDAP/Active Directory](plugins/ldap-auth.md) authentication with group mapping. |
| Least privilege | Granular per-user and per-directory permissions (list, download, upload, overwrite, delete, create directories, rename, symlink, chmod). Users are restricted to their home directory. |
| Role-based access | [Groups](groups.md) for policy inheritance, [Roles](roles.md) for delegated administration. |
| IP-based restrictions | Per-user and global IP allow/deny lists. [GeoIP filtering](plugins/geoip-filter.md) by country. Access time restrictions per user. |
| Password policies | Configurable password strength requirements (entropy, length, character classes) at the global or [group](groups.md) level. Automatic password expiration with configurable thresholds. |

## Threat protection

| Requirement | SFTPGo capability |
| ----------- | ----------------- |
| Brute-force protection | Built-in [defender](defender.md) with configurable scoring and automatic IP banning. |
| Rate limiting | Per-protocol and per-IP [rate limiting](rate-limiting.md) with defender integration. |
| Antivirus / DLP | [ICAP integration](filesystem-actions.md#icap) for scanning uploaded files with antivirus and data loss prevention systems. Quarantine support via virtual folders. |
| Secure file sharing | Public shares with password protection, email-based authentication, expiration, download limits, and IP restrictions. Administrators can enforce share policies and restrict shareable paths via [groups](groups.md). |

## Data governance

| Requirement | SFTPGo capability |
| ----------- | ----------------- |
| Data retention | Automated [retention policies](tutorials/eventmanager-retention.md) with per-directory rules. Expired files can be deleted or archived. |
| Data isolation | Per-user home directories with strict filesystem isolation. [Virtual folders](virtual-folders.md) with independent quotas. |
| Quota management | Per-user and per-folder quotas (total size and/or file count). Bandwidth throttling with separate upload/download limits. |
| Automated workflows | [Event Manager](eventmanager.md) for automated responses to file operations: copy, compress, encrypt (PGP), notify, and more. |

## Infrastructure

| Requirement | SFTPGo capability |
| ----------- | ----------------- |
| High availability | Multi-node [clustering](features.md#high-availability-and-deployment) with near real-time configuration propagation. |
| Monitoring | [Prometheus metrics](metrics.md) for alerting and dashboards, including per-user transfer metrics. Structured JSON logs for integration with log management systems. |
| Infrastructure as Code | [Terraform provider](https://registry.terraform.io/providers/drakkan/sftpgo/latest){:target="_blank"} for declarative management of users, groups, folders, and event rules. [REST API](rest-api.md) with JWT and API key authentication. |
| Deployment flexibility | Runs on Linux, Windows, Docker, and [Kubernetes](docker.md#kubernetes). Available on [AWS](tutorials/sftpgo-aws.md), [Azure](installation.md#azure), and [Google Cloud](tutorials/sftpgo-google-cloud.md) marketplaces. |

:information_source: The features listed above describe the technical capabilities available in SFTPGo. Achieving compliance with any specific framework depends on how the software is configured and operated within your environment. For SFTPGo's managed [SaaS offering](https://sftpgo.com/saas){:target="_blank"}, infrastructure-level controls are managed by SFTPGo.
