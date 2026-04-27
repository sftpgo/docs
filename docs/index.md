---
description: "SFTPGo — open-source managed file transfer (MFT) solution with SFTP, FTP/S, WebDAV, HTTPS. Cloud storage backends, event-driven automation, multi-factor authentication, OIDC SSO."
---

# What is SFTPGo?

SFTPGo is a fully featured, open-source managed file transfer (MFT) solution. It provides secure file exchange over SFTP, SCP, FTP/S, WebDAV, and a built-in HTTPS WebClient, with support for local and cloud storage backends.

SFTPGo is built around a few core principles:

- **Protocol-agnostic access.** Users can connect via SFTP, SCP, FTP/S, WebDAV, or the built-in web interface — the same files and permissions apply across all protocols.
- **Storage abstraction.** Local filesystem, encrypted filesystem, S3-compatible storage, Google Cloud Storage, Azure Blob, and remote SFTP servers are all supported as storage backends, including within the same installation.
- **Event-driven automation.** The [Event Manager](eventmanager.md) allows administrators to define rules that react to file operations, provider changes, or schedules — enabling automated workflows such as notifications, cross-backend file transfers, data retention, and more.
- **Security and access control.** Built-in [brute-force protection](defender.md), Geo-IP filtering (via plugin), per-protocol [rate limiting](rate-limiting.md), and fine-grained per-user/per-directory permissions. Authentication supports passwords, public keys, SSH certificates, multi-factor (TOTP), LDAP/Active Directory (via plugin), and [OpenID Connect](oidc.md) for SSO.

SFTPGo runs everywhere: from small embedded devices to large clusters, on Linux, Windows, macOS and FreeBSD, on x86, ARM, and ppc64 architectures. Configuration data can be stored in SQLite, MySQL, PostgreSQL, CockroachDB, Bolt, or in-memory. A [Terraform provider](https://registry.terraform.io/providers/drakkan/sftpgo/latest){:target="_blank"} is available for Infrastructure as Code workflows.

The WebAdmin interface provides centralized management for users, groups, virtual folders, event rules, and server configuration. The [WebClient](web-interfaces.md#webclient) gives end users a browser-based file manager with credential management, two-factor authentication, and secure file sharing.

**Ready to get started?** See the [installation guide](installation.md) and then the [Getting Started](initial-configuration.md) walkthrough. For a complete feature list, see [Features](features.md).

![Architectural overview](assets/img/sftpgo%20architecture.png#only-light){data-gallery="architecture"}
![Architectural overview](assets/img/sftpgo%20architecture-dark.png#only-dark){data-gallery="architecture"}

## Licensing

SFTPGo source code is licensed under [GNU AGPL-3.0-only](https://www.gnu.org/licenses/agpl-3.0.en.html){:target="_blank"} with [additional terms](https://github.com/drakkan/sftpgo/blob/main/NOTICE){:target="_blank"}.

The [theme](https://keenthemes.com/products/templates-mega-bundle){:target="_blank"} used in WebAdmin and WebClient user interfaces is proprietary, this means:

- KeenThemes HTML/CSS/JS components are allowed for use only within the SFTPGo product and restricted to be used in a resealable HTML template that can compete with KeenThemes products anyhow.
- The SFTPGo WebAdmin and WebClient user interfaces (HTML, CSS and JS components) based on this theme are allowed for use only within the SFTPGo product and therefore cannot be used in derivative works/products without an explicit grant from the [SFTPGo Team](mailto:support@sftpgo.com).

**Note:** This content is provided for informational purposes only and does not constitute legal advice. If you have specific questions about license compliance or whether your particular use case is permitted, we recommend consulting your legal team.

## Copyright

Copyright (C) 2019 - 2026 Nicola Murino
