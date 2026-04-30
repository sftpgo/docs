---
description: "SFTPGo Event Manager: automate file transfer workflows with rules triggered by uploads, downloads, schedules, and provider changes."
---

# Event Manager

The Event Manager allows administrators to define automated responses to events occurring within SFTPGo — file uploads, user creation, scheduled tasks, and more. It is composed of two elements: **rules** and **actions**.

- A **rule** defines *when* something should happen: it specifies which events to react to, optional filters to narrow the scope, and which actions to execute.
- An **action** defines *what* to do: send a notification, run a command, scan a file, copy it to another location, and so on.

Actions support dynamic **placeholders** — variables like `{{.Name}}`, `{{.VirtualPath}}`, or `{{.FileSize}}` that are replaced at runtime with contextual data from the triggering event. A rich set of **helper functions** is also available for formatting and transforming values inside templates. See the [Placeholders & Templates](placeholders.md) reference for the full list.

:bulb: **Looking for hands-on walkthroughs?** The Tutorials section contains end-to-end, copy-pasteable Event Manager recipes — start with [the Event Manager tutorial overview](tutorials/eventmanager.md) and the topic-specific walkthroughs that follow:

- [Daily Backups](tutorials/eventmanager-backup.md)
- [Automatic Folder Structure](tutorials/eventmanager-auto-dirs.md)
- [Upload Notifications & Webhooks](tutorials/eventmanager-notifications.md) — Slack, Microsoft Teams, Discord, Google Chat, Mattermost
- [Auto Provisioning via IdP](tutorials/eventmanager-idp.md)
- [Data Retention](tutorials/eventmanager-retention.md)
- [Copy & Archive Workflows](tutorials/eventmanager-copy.md)
- [Antivirus Scanning (ICAP)](tutorials/eventmanager-icap.md)
- [Upload Approval Workflow](eventmanager-approval.md)
- [PGP Encryption & Decryption](tutorials/eventmanager-pgp.md)
- [Virtual Folders Integration](tutorials/eventmanager-folders.md)
- [Recycle Bin](tutorials/eventmanager-recycle-bin.md)

## Rules

A rule is built around a triggering event. When the event occurs and all conditions are met, the associated actions are executed. The following trigger types are supported:

| Trigger | Description |
| --------- | ------------- |
| **Filesystem events** | Reacts to file operations: `upload`, `download`, `delete`, `rename`, `mkdir`, `rmdir`, `copy`, `ssh_cmd`. |
| **Provider events** | Reacts to data provider changes: `add`, `update`, `delete` on users, admins, groups, folders, shares, and other resources. |
| **Schedule** | Runs on a cron schedule. The scheduler uses **UTC** time. |
| **IP Blocked** | Fires when the [Defender](./defender.md) blocks an IP address. |
| **Certificate** | Fires when a TLS certificate is renewed via the built-in ACME protocol (both success and failure). |
| **On demand** | Triggered manually from the WebAdmin or via the REST API. |
| **Identity Provider login** | Fires when a user or admin logs in through an external Identity Provider. |

### Conditions

You can narrow the scope of a rule by adding conditions. For example, you can react to uploads only from a specific user, or only when the file matches a path pattern.

Conditions are available for usernames, roles, groups, protocols, and file paths (shell-like glob patterns).

#### Inverse match

Each name, group, role, or path filter entry has an **Inverse match** toggle. When enabled, the pattern means "match everything *except* this". Use it to express exclusions without having to list every allowed value — for example, a rule that should run for everyone except internal users can be scoped with a single group filter for the `internal` group with inverse match enabled. Inverse match applies independently to each entry, so you can combine positive and negative patterns in the same filter.

#### Name, group, and role filters

For rules triggered by a schedule or by provider events — where SFTPGo must resolve the filters to the list of users the action applies to — prefer **exact values** over wildcard patterns when you can. With exact filters SFTPGo fetches only the matching users; with a wildcard pattern (`*`, `?`, `[...]`) it has to load every user and check them one by one, which gets expensive on instances with many users.

The wildcard penalty is the same on any of the three axes: a single wildcard anywhere in the rule conditions is enough to trigger the full scan. Exact values on names, groups, or roles are equivalent — pick the one that expresses the intent most clearly (e.g. a group name when the target set already maps to a group).

For small installations the difference is negligible. On instances with thousands of users, prefer exact names, groups, or roles whenever the target set is known up front, instead of patterns like `customer_*`.

Filters on filesystem events (uploads, downloads, etc.) do not have this cost — the triggering user is already known, so the match is always a single in-memory check regardless of whether the filter is exact or a pattern.

#### Path filters

By default, path patterns match against the **virtual path** — the normalized, backend-independent path that clients see (e.g., `/uploads/file.txt`).

Each pattern can optionally be set to **match on filesystem path** instead. This matches against the actual backend storage path, which is useful when you need to filter by physical location, bucket, or Windows drive letter.

Filesystem paths are backend-dependent:

| Backend | Example filesystem path |
| --------- | ------------------------ |
| Local (Linux) | `/home/sftpgo/user1/uploads/file.txt` |
| Local (Windows) | `C:/Users/sftpgo/user1/uploads/file.txt` |
| UNC (Windows) | `//server/share/uploads/file.txt` |
| S3 | `prefix/uploads/file.txt` |
| Azure Blob | `prefix/uploads/file.txt` |
| GCS | `prefix/uploads/file.txt` |
| SFTPFs | `/remote/path/uploads/file.txt` |

:information_source: **Always use forward slashes** (`/`) in patterns, even for Windows paths. Backslashes are automatically converted to forward slashes when saving the pattern.

:warning: **Matching is case-sensitive** on all platforms, including Windows. Use character classes if needed (e.g., `[Dd]ata`).

:warning: **Cloud storage paths have no leading slash**, so a pattern like `/**/*.exe` will not match them. Use `**/*.exe` instead.

You can mix virtual path and filesystem path patterns in the same rule. Each pattern is evaluated against its configured path type independently.

### Action execution order and options

Actions within a rule are executed **sequentially**, in the order they are listed. For each action, you can configure the following options:

- **Stop on failure** — If this action fails, skip all remaining actions in the rule.
- **Failure action** — Mark an action so that it only executes when a previous (non-failure) action has failed. This is useful for error notifications. Note: a failure action runs when *another action in the rule* fails, not when the triggering event itself fails (e.g., a failed download still runs the main action, not the failure action).
- **Execute sync** — For upload events, execute the action synchronously: the client waits for the action to complete before receiving a response. Required for pre-* events (pre-delete, pre-download, pre-upload). If pre-* sync actions succeed, the operation is allowed; otherwise the client receives a permission denied error. Be mindful of client timeouts for long-running actions.
- **Execute before file publish** — For upload events with atomic uploads enabled, run the action on the temporary file *before* it is renamed to its final path. The file remains invisible to other users during processing. On failure, the temporary file is deleted — the file never becomes visible. Useful for antivirus scanning (ICAP), content validation, or any processing that must complete before the file is accessible. Can be combined with "Execute sync" for client-side feedback. See [Execute Before File Publish](execute-before-file-publish.md) for details.

If you run multiple SFTPGo instances connected to the same data provider, you can choose whether to allow simultaneous execution for scheduled rules.

### Compatibility notes

Some actions are not available for certain trigger types. Rules containing incompatible actions are silently skipped at runtime:

- **Filesystem events** — Folder quota reset is not supported (there is no direct way to determine the affected folder).
- **Provider events** — User quota reset, transfer quota reset, data retention check, and filesystem actions are only supported when a user is updated (they execute for the affected user). Folder quota reset is only supported for folders. Filesystem actions are not executed on user deletion (the user no longer exists when actions run).
- **IP Blocked** / **Certificate** — User quota reset, folder quota reset, transfer quota reset, data retention check, and filesystem actions are not supported.
- **Email with attachments** — Supported for filesystem events and provider events involving a user add/update (a user context is needed to read attached files).
- **HTTP multipart with file attachments** — Same restriction as email with attachments.

## Actions

The following action types are available. Actions marked with details links have dedicated documentation pages.

### Notifications

| Action | Description |
| -------- | ------------- |
| **HTTP notification** | Send an HTTP/S request (GET, POST, PUT, DELETE) to an external endpoint. Supports custom headers, query parameters, and a request body. Placeholders are supported in the body, headers, and query parameter values. For POST/PUT, you can also send files as multipart attachments. |
| **Email notification** | Send a plain-text email. Placeholders are supported in recipients, subject, and body. File attachments from the user's filesystem are supported (up to 10 MB total). Requires SMTP configuration. |
| **Command execution** | Run a system command with parameters passed as environment variables. Placeholders are supported for environment variable values. :warning: Command execution is disabled by default for security reasons and must be explicitly enabled. |

### Maintenance

| Action | Description |
| -------- | ------------- |
| **Backup** | Save a full data provider backup (users, folders, groups, admins, etc.) to the configured backup directory. The filename includes the day of the week, hour, and minute. |
| **Rotate log file** | Rotate the current log file regardless of its size (only when file-based logging is enabled). |
| **User quota reset** | Recalculate the disk quota usage for matching users based on actual storage consumption. |
| **Folder quota reset** | Recalculate the disk quota usage for matching virtual folders. |
| **Transfer quota reset** | Reset the transfer quota counters to zero for matching users. |

### Lifecycle checks

| Action | Description |
| -------- | ------------- |
| **Data retention check** | Apply per-folder retention policies. Files older than the configured threshold are automatically deleted or archived. |
| **Password expiration check** | Send email notifications to users whose passwords are about to expire. |
| **User expiration check** | Generate notifications listing expired user accounts. |
| **User inactivity check** | Detect and optionally disable or delete users who have been inactive beyond a configured threshold. |
| **Share expiration check** | Manage the lifecycle of shares based on inactivity, expiration date, or maximum token usage. Supports configurable inactivity thresholds, advance notice periods (to trigger warnings before expiration), and grace periods (soft delete). Enable **Split events** to generate individual events per share — useful for sending 1-to-1 notifications. For group shares, warnings are sent to all members; the actual deletion event targets the share owner. |

### Filesystem actions

Filesystem actions let you manipulate files and directories as part of an event rule. SFTPGo grants the required permissions automatically — the behavior is equivalent to performing the same operations from an SFTP client, with the same restrictions.

See the [Filesystem Actions](filesystem-actions.md) page for a complete reference with detailed options for each action type.

Available operations: **Rename**, **Delete**, **Create directories**, **Path exists**, **Copy** (with source disposition, glob patterns, retries, and continue-on-error), **Compress** (ZIP), **Extract** (ZIP with security limits), **PGP** encryption/decryption with optional signing, **Metadata Check** (cloud backends), **IMAP** (fetch email attachments), **ICAP** (antivirus/DLP scanning).

### Integration and reporting

| Action | Description |
| -------- | ------------- |
| **Identity Provider account check** | Automatically create or update user/admin accounts when someone logs in through an external Identity Provider. |
| **Event report** | Query and aggregate stored filesystem events, grouped by user. Designed for scheduled rules that send periodic digest notifications. See the [Event Report](event-report.md) page for details. |

## Virtual folders

Virtual folders can be combined with filesystem actions to operate across storage backends or outside a user's security context. Two folder options are available on filesystem actions:

- **Source folder** — Overrides the filesystem used to read source files. By default, actions operate on the triggering user's filesystem. Specifying a source folder lets you read from a different location — essential for scheduled tasks and advanced workflows where no user context is available.
- **Target folder** — Overrides the filesystem used to write target files. This enables cross-backend operations such as copying uploads to a different S3 bucket, archiving files to an external SFTP server, or accessing restricted areas of the same storage backend.

When **both** a source folder and a target folder are specified, the action runs as a **system action** — it executes once (not per-user) with a special system identity, regardless of how many users match the rule's conditions. When only one folder is specified, the action runs per-user as usual.

See the [Virtual Folders tutorial](tutorials/eventmanager-folders.md) for practical examples of virtual folder integration.
