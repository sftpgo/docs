---
description: "Event Manager tutorials for SFTPGo: automate backups, notifications, data retention, antivirus scanning, and file transfer workflows."
---

# Event Manager Tutorials

The Event Manager enables administrators to automate server operations by configuring HTTP notifications, executing commands, sending email alerts, and more — based on server events or scheduled triggers.

At its core, the Event Manager consists of two main components: **rules** and **actions**.

- **Rules** define the conditions that determine when an action should be executed. Think of a rule as a "when this happens, and these conditions are met, then do that" type of logic.
- **Actions** are the tasks carried out when a rule is triggered. These actions can be dynamically customized using placeholders — variables that represent contextual data related to the event (such as file name, username, or file size). To further tailor these values, SFTPGo provides helper functions that format or transform placeholders directly within your templates. For a complete list of placeholders and helper functions, see the [Placeholders & Templates](../placeholders.md) reference.

## Tutorials

The following step-by-step tutorials cover common use cases:

- [**Daily Backups**](eventmanager-backup.md) — Schedule automatic data backups with email notifications.
- [**Automatic Folder Structure**](eventmanager-auto-dirs.md) — Create a standard folder structure for new users.
- [**Upload Notifications**](eventmanager-notifications.md) — Receive email notifications after uploads, and send custom data to external webhooks.
- [**Auto Provisioning via Identity Provider**](eventmanager-idp.md) — Automatically create user accounts when they log in through an external IdP.
- [**Data Retention**](eventmanager-retention.md) — Automatically delete or archive files older than a configurable threshold.
- [**Copy & Archive Workflows**](eventmanager-copy.md) — Copy files with glob patterns, source disposition, and cross-backend transfers.
- [**Antivirus Scanning (ICAP)**](eventmanager-icap.md) — Scan uploaded files before they become visible, using ICAP with Execute Before File Publish.
- [**PGP Encryption and Decryption**](eventmanager-pgp.md) — Automatically encrypt or decrypt files after upload using PGP keys.
- [**Virtual Folders Integration**](eventmanager-folders.md) — Copy files across storage backends and outside a user's security context.
- [**Recycle Bin**](eventmanager-recycle-bin.md) — Move files to a recycle folder instead of deleting them permanently.

## SMTP Configuration

Many tutorials use email actions. Make sure you have a working SMTP configuration. You can configure it using environment variables:

```shell
SFTPGO_SMTP__HOST="your smtp server host"
SFTPGO_SMTP__FROM="SFTPGo <sftpgo@example.com>"
SFTPGO_SMTP__USER=sftpgo@example.com
SFTPGO_SMTP__PASSWORD="your password"
SFTPGO_SMTP__AUTH_TYPE=1 # change based on what your server supports
SFTPGO_SMTP__ENCRYPTION=2 # change based on what your server supports
```

:information_source: The SMTP server can also be configured directly through the WebAdmin UI by navigating to **Server Manager > Configurations > SMTP**.
