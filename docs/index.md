# What is SFTPGo?

SFTPGo is a managed, event-driven file transfer solution that abstracts storage backends and provides access to files through its built-in WebClient over HTTPS, as well as via standard SFTP, SCP, FTP/S, and WebDAV protocols.

With SFTPGo you can leverage local and cloud storage backends for exchanging and storing files internally or with business partners using the same tools and processes you are already familiar with.

The WebAdmin UI allows to easily create and manage your users, folders, groups and other resources.

The WebClient UI allows end users to change their credentials, browse and manage their files in the browser and setup two-factor authentication which works with Microsoft Authenticator, Google Authenticator, Authy and other compatible apps.

SFTPGo in short:

- Event driven file transfer solution.
- Multiple protocols: SFTP, SCP, FTP/S, WebDAV, HTTP/S, REST API.
- Multiple storage backends: S3 Compatible, Google Cloud Storage, Azure Blob, other SFTP servers, local filesystem, encrypted local filesystem.
- Multiple data providers: SQLite, MySQL, PostresSQL, CockroachDB, Bolt and in-memory.
- Extensible via plugins and hooks.
- It works everywhere: on small embedded devices or large Kubernetes clusters. On Linux, Windows, macOS, FreeBSD. On x86, arm, ppc64.

![Architectural overview](assets/img/sftpgo%20architecture.png){data-gallery="architecture"}

## Enterprise edition

This documentation applies to the Enterprise edition of SFTPGo. If you're using the Open Source edition, please refer to the [corresponding documentation](/latest/){:target="_blank"}.

SFTPGo Enterprise is an enhanced version of the open-source SFTPGo, based on its core functionality, and tailored for organizations that require more advanced features, improved performance and a commercially licensed solution.

New features are regularly added to the Enterprise edition, which may or may not be backported to the open-source edition.

Key Enhancements:

- Significant performance improvements for cloud storage backends — up to 70% faster when transferring many small files.
- A more powerful EventManager, enabling advanced and flexible automation workflows.
- PGP encryption and decryption.
- WOPI protocol integration, allowing in-browser document editing and real-time collaboration directly within the SFTPGo WebClient. Multiple users can edit the same document simultaneously, with live updates for all participants.
- Email-based authentication for public shares, enhancing security and access control.
- Numerous additional customization options and configuration improvements across the platform.
- Support included.

The open-source version of SFTPGo will continue to be maintained and updated with bug fixes and improvements, ensuring both versions remain reliable and functional.

## Licensing

The Enterprise version is offered under a [proprietary license](https://sftpgo.com/assets/LICENSE.pdf){:target="_blank"} that removes the restrictions of the open-source AGPLv3.

## Copyright

Copyright (C) 2019 - 2025 Nicola Murino
