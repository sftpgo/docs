---
description: "SFTPGo — managed file transfer (MFT) platform with SFTP, FTP/S, WebDAV, HTTPS. HA clustering, Kubernetes, Terraform, SSO, cloud storage, automation, audit logging."
---

# What is SFTPGo?

SFTPGo is a fully featured, open-core managed file transfer (MFT) platform. It provides secure file exchange over SFTP, SCP, FTP/S, WebDAV, and a built-in HTTPS WebClient, with support for local and cloud storage backends.

SFTPGo is built around a few core principles:

- **Protocol-agnostic access.** Users can connect via SFTP, SCP, FTP/S, WebDAV, or the built-in web interface — the same files and permissions apply across all protocols.
- **Storage abstraction.** Local filesystem, encrypted filesystem, S3-compatible storage, Google Cloud Storage, Azure Blob, and remote SFTP/FTP servers are all supported as storage backends, including within the same installation.
- **Event-driven automation.** The [Event Manager](eventmanager.md) allows administrators to define rules that react to file operations, provider changes, or schedules — enabling automated workflows such as notifications, cross-backend file transfers, antivirus scanning, data retention, PGP encryption, and more.
- **Security and compliance.** Data-at-rest encryption, [audit logging](plugins/audit-logs.md), brute-force protection, geo-IP filtering, post-quantum cryptography for SSH and HTTPS, and fine-grained access controls help meet regulatory requirements. Authentication supports passwords, public keys, certificates, multi-factor, LDAP/Active Directory, and OpenID Connect (SSO).

SFTPGo is designed for high availability: it supports multi-node clustering with near real-time configuration propagation, runs natively on Kubernetes (with an official Helm chart), and is available on the AWS, Azure, and Google Cloud marketplaces. A [Terraform provider](https://registry.terraform.io/providers/drakkan/sftpgo/latest){:target="_blank"} is available for Infrastructure as Code workflows.

The WebAdmin interface provides centralized management for users, groups, virtual folders, event rules, and server configuration. The WebClient gives end users a browser-based file manager with credential management, two-factor authentication, and secure file sharing.

**Ready to get started?** See the [installation guide](installation.md) and then the [Getting Started](initial-configuration.md) walkthrough. For a complete feature list, see [Features](features.md).

**Using an AI assistant?** The project publishes an AI-agent-agnostic skill that teaches Claude Code, Cursor, Copilot, ChatGPT, Gemini, and compatible models how to produce correct SFTPGo configuration, REST API payloads, and Event Action templates. See [AI Assistants](ai-assistants.md) for install instructions.

![Architectural overview](assets/img/sftpgo%20architecture.png#only-light){data-gallery="architecture"}
![Architectural overview](assets/img/sftpgo%20architecture-dark.png#only-dark){data-gallery="architecture"}

## Open Source and Enterprise editions

SFTPGo is available in two editions:

- The **Open Source edition** is released under the AGPLv3 license. It includes the full protocol stack, storage backends, WebAdmin and WebClient interfaces, basic Event Manager automation, OpenID Connect support, and the REST API. Documentation for the Open Source edition is available [here](/latest/){:target="_blank"}.
- The **Enterprise edition** is offered under a [proprietary license](https://sftpgo.com/assets/LICENSE.pdf){:target="_blank"} that removes the AGPLv3 restrictions. It includes commercial support and additional features.

This documentation covers the Enterprise edition.

### What Enterprise adds

| Area | Enterprise additions |
| ------ | --------------------- |
| **Event Manager** | Full [template engine](placeholders.md) with helper functions, conditions, and loops. [Virtual folder integration](eventmanager.md#virtual-folders) for cross-backend operations. Additional actions: [ICAP](filesystem-actions.md#icap) (antivirus/DLP), [IMAP](filesystem-actions.md#imap) (email ingestion), [event reports](event-report.md), PGP encryption/decryption. [Execute Before File Publish](execute-before-file-publish.md) for staged upload processing. Enhanced [copy action](filesystem-actions.md#copy) with source disposition, glob patterns, and retries. Data retention with [archival](tutorials/eventmanager-retention.md). |
| **OpenID Connect** | Configurable [role mapping](oidc.md#role-mapping), [PKCE without client secret](oidc.md#pkce-without-client-secret), session control (`max_age`, `prompt`), Azure B2C compatibility, customizable login labels. |
| **WebClient** | [WOPI](initial-configuration.md#document-editing-and-collaboration) document editing and real-time collaboration. [TUS](https://tus.io/){:target="_blank"} resumable uploads for reliability through proxies/CDNs. |
| **Sharing** | Email-based authentication. Group-based delegation and [governance policies](tutorials/shares.md). [Path and scope restrictions](tutorials/shares.md#restricting-shareable-paths). |
| **Storage** | Optimized cloud backend performance (up to 70% faster for small files). In-memory transfers (no local temp storage). GCS [Hierarchical Namespace](google-cloud-storage.md). SFTP backend [SOCKS proxy](sftpfs.md). [FTP as storage backend](ftpfs.md). |
| **Administration** | [Clustering](features.md#administration-and-operations) with near real-time propagation. Extended [WebAdmin configuration](features.md#web-interfaces): OIDC, LDAP, Geo-IP, TLS certificates, email templates, SSH host keys — all from the UI. [API key management](rest-api.md#api-key-authentication) from the UI. |

Both editions are actively maintained. The Open Source edition receives regular bug fixes and improvements. New features are developed primarily for the Enterprise edition and may be backported to the Open Source edition over time.

## Copyright

Copyright (C) 2019 - 2026 Nicola Murino
