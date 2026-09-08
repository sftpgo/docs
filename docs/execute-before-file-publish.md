---
description: "Run actions on uploaded files before they become visible in SFTPGo. Use for antivirus scanning, content validation, or approval workflows."
---

# Execute Before File Publish

## Overview

SFTPGo supports running actions on uploaded files **before** they are published at their final path. This feature is designed for use cases where files must be verified, scanned, or processed before being published — such as antivirus scanning, content validation, DLP checks, or format conversion.

When enabled, an upload is written to a temporary file while the configured actions run. If all actions succeed, the file is renamed to its final path. If any action fails, the temporary file is removed, or quarantined by the action, and the file never appears at its final path.

Two modes are available:

- **Asynchronous** ("Execute before file publish" only): the client receives a success response immediately without waiting for the actions to complete. Use it for long-running actions where client timeout is a concern.
- **Synchronous** ("Execute before file publish" + "Synchronous execution"): the client waits for the actions to complete and receives success or failure. Use it when the client needs immediate feedback and timeout is acceptable.

In both modes the file is not at its final path until all actions succeed. The temporary file is an ordinary file in the destination directory: configure the [file pattern filter](#step-2-configure-file-pattern-filter) below to hide it from the user and from anyone else who reaches that directory.

## How It Works

### Asynchronous mode

1. The client uploads a file via SFTP, FTP, or HTTP/WebDAV.
2. The file is written to a temporary file (`.sftpgo-upload.<id>.<filename>`) in the same directory as the final destination.
3. The client receives a success response — the upload is complete from their perspective.
4. The configured actions run on the temporary file in the background (e.g., antivirus scan via ICAP).
5. If all actions pass: the file is renamed to its final path.
6. If any action fails: the temporary file is removed, or quarantined by the action, and the file never appears at its final path.

### Synchronous mode

1. The client uploads a file via SFTP, FTP, or HTTP/WebDAV.
2. The file is written to a temporary file (`.sftpgo-upload.<id>.<filename>`) in the same directory as the final destination.
3. The configured actions run on the temporary file — the client waits.
4. If all actions pass: the file is renamed to its final path, the client receives a success response.
5. If any action fails: the temporary file is removed, or quarantined by the action, and the client receives an error response.

## Requirements

- **Atomic uploads must be enabled.** Set the environment variable `SFTPGO_COMMON__UPLOAD_MODE=1`. Mode `2` (atomic with resume support) also stages uploads, but it keeps interrupted uploads at their final path without running the actions: use mode `1`.
- **A file pattern filter that hides the temporary files.** Without it, the temporary file appears in directory listings and can be downloaded, renamed or deleted like any other file, by the uploader and by every user who reaches that directory, while the actions run. With a deny pattern for `.sftpgo-upload*` and the **Hide** policy the temporary file is invisible to users; the configured actions read it regardless, since [event actions are not bound by file pattern filters](eventmanager.md#how-an-action-is-authorized).

## Configuration

### Step 1: Enable Atomic Uploads

Set the following environment variable:

```shell
SFTPGO_COMMON__UPLOAD_MODE=1
```

### Step 2: Configure File Pattern Filter

For each user or group, add a file pattern filter to block access to temporary upload files.

File pattern filters are defined under the user's or group's **Files / Patterns** section and apply to a specific virtual path. Each entry has four fields:

- **Path** — the virtual path where the filter applies, as seen by the user (**not** a filesystem path). A filter set on `/` applies everywhere (every directory and subdirectory in the user's home); a filter on `/inbox` applies only to `/inbox` and its subdirectories unless a more specific filter overrides it. To cover temporary upload files regardless of the destination directory, set this to `/`.
- **Allowed patterns** — shell-like (glob) patterns for files that are allowed. Evaluated before denied patterns. Matching is case-insensitive. Leave empty if you do not want to restrict uploads to specific extensions.
- **Denied patterns** — shell-like (glob) patterns for files that are blocked. Matching is case-insensitive. Add `.sftpgo-upload*` here to block access to staging files.
- **Deny policy** — controls the behavior for denied files:
  - **Default (0)** — denied files appear in directory listings, but all operations on them (download, upload, delete, rename, overwrite) are blocked.
  - **Hide (1)** — same restrictions as the default policy, plus denied files are hidden from directory listings entirely. Use this option for temporary upload files so they never appear to the client.

For the typical setup, add one entry with:

| Field | Value |
| ---------- | ------- |
| Path | `/` |
| Allowed patterns | *(leave empty)* |
| Denied patterns | `.sftpgo-upload*` |
| Deny policy | Hide |

The filter applies to all directories where uploads can occur, so you do not need to add a separate entry per directory. Set it on a group to cover every member at once.

:information_source: These patterns use shell-style globbing (`*`, `?`, character classes), not regular expressions. The leading dot in `.sftpgo-upload*` matches the exact prefix used by SFTPGo for staging files.

### Step 3: Create an Event Action

Create an action that will process the uploaded file. Supported action types include:

- **ICAP** — Built-in antivirus/DLP scanning. Supports every storage backend (local, encrypted, S3, Azure, GCS, SFTP).
- **HTTP** — Call an external webhook for custom validation.
- **Command** — Run a local script or program.
- **Any other action type** — Email notifications, filesystem operations, etc.

### Step 4: Create an Event Rule

Create a rule with:

- **Trigger:** Filesystem events
- **Events:** Upload
- **Actions:** Select your action and enable the **"Execute before file publish"** option

You can enable **both** "Execute before file publish" and "Synchronous execution" on the same action. This gives you:

- **Consumer protection:** the file is not published during processing (it stays at the temporary path)
- **Synchronous feedback:** the client waits for the action to complete and receives success or failure

The trade-off is potential client timeout for large files — the client blocks until the action completes. Without "Synchronous execution", the client receives an immediate success response and the action runs in the background.

## Using the Built-in ICAP Action

The ICAP action sends the uploaded file to an ICAP server for antivirus or DLP scanning. It:

- Reads files through the storage layer, so it supports every backend
- Lets you choose what happens to the file for each verdict: delete it, quarantine it (via virtual folders, also on a different storage backend), overwrite it with the sanitized version returned by the server, or leave it in place. See [ICAP](filesystem-actions.md#icap) for the settings
- Works with servers implementing the ICAP protocol (RFC 3507)

Combined with "Execute before file publish", a file the server does not accept is never published, whatever the chosen option: the temporary file is deleted or quarantined by the action, or removed by SFTPGo. The **Overwrite** adapt action publishes the sanitized version in its place.

## Storage Backend Support

| Backend | Supported | Notes |
| --------- | ----------- | ------- |
| Local filesystem | Yes | Files staged in the same directory |
| Encrypted filesystem (CryptFs) | Yes | ICAP scans the decrypted content |
| AWS S3 | Yes | Rename uses server-side copy + delete |
| Azure Blob Storage | Yes | Rename uses server-side copy + delete |
| Google Cloud Storage | Yes | Rename uses server-side copy + delete |
| SFTP (remote) | Yes | When buffer size is 0 (direct streaming) |
| FTP (remote) | Yes | Rename via RNFR/RNTO |

:information_source: Renaming on cloud storage involves a server-side copy followed by a delete. For very large files, this adds latency proportional to the file size. This overhead is in addition to the action processing time. On S3, Azure Blob, Google Cloud Storage and FTP backends staging is enabled as soon as one rule with "Execute before file publish" exists: every upload on these backends, by any user and whether or not it matches the rule, goes through the temporary file and the rename.

## Client Behavior

### Async mode (Execute before file publish only)

- **Upload response:** The client receives a success response immediately after the upload completes. The client does not wait for actions to finish.
- **Failed actions:** If an action fails, the temporary file is removed or quarantined. The client has already received a success response — there is no way to notify the client of the failure through the protocol. Use email or webhook actions to alert administrators.

### Sync mode (Execute before file publish + Synchronous execution)

- **Upload response:** The client blocks until all staged actions complete. If all succeed, the file is renamed and the client receives a success response. If any fails, the temporary file is removed or quarantined and the client receives an error.
- **Timeout risk:** For large files (e.g., 2 GB antivirus scan taking 90-120 seconds), the client may timeout. Configure client-side timeouts accordingly.

### The file while the actions run

Until the actions complete and the file is renamed:

- **The final path does not exist.** It is absent from directory listings, and download, rename and delete on it fail with "file not found". This applies to every user and every connection, including the uploader's.
- **Size checks from the uploading session work on SFTP and FTP.** Many clients verify an upload by requesting the attributes of the file they just sent. Within the SFTP or FTP session that performed the upload, such a request on the final path returns the size and modification time of the uploaded file while the actions run, so these clients report success. Any other session, and any other user, sees the final path as missing.
- **WebDAV and HTTP verification requests report the file as missing.** These protocols send each request on its own connection, so a check performed after the upload (for example a `PROPFIND`) does not find the file until the actions complete; clients that verify uploads may report an error or retry. Use sync mode for these protocols.
- **The temporary file** (`.sftpgo-upload.*`) is in the destination directory. With the file pattern filter of step 2 it is hidden from users and every operation on it is blocked; without the filter it is listed and accessible like any other file. The configured actions read it in either case.

## Interaction between staged and non-staged rules

When multiple rules match the same upload event, some with "Execute before file publish" and some without, the behavior is:

1. Rules with "Execute before file publish" execute **first**, on the temporary file (before rename).
2. If all staged actions succeed, the file is renamed to its final path.
3. Rules **without** "Execute before file publish" (standard sync or async actions in separate rules) execute **after** the rename, on the file at its final path — the file is visible at this point.
4. If any staged action fails, the file is never renamed. The upload is then reported to the other rules as a **failed** upload, like an upload interrupted by the client: a rule whose **Status filters** select the successful status does not run, a rule without a status filter runs with the failed status and finds nothing at the final path. Set **Status filters** to the successful status on rules that must react to published files alone.

:warning: Standard sync actions in separate rules do **not** delay the uploader's response when staged actions are present. In async mode, the client receives OK immediately; in sync mode, only the staged actions block the client. In both cases, standard actions in other rules run after the rename, in the background. If you need an action to block the client AND run before the file is visible, use "Execute before file publish" combined with "Synchronous execution" on the same action in the same rule.

## Combining with Other Actions

You can use "Execute before file publish" alongside standard post-upload actions by placing them in **separate rules**:

- **Separate rules:** Actions in other rules (synchronous or asynchronous) execute **after** the file is published (renamed to its final path). The file is visible during the execution of these actions. If the staged action fails, they receive the upload as failed, as described above.
- **Same rule:** A rule with "Execute before file publish" actions can only contain other staged actions and failure actions. Non-staged actions (webhooks, email notifications, etc.) must be placed in a separate rule. This is enforced by validation.
- **Failure actions:** Failure actions in the same rule as staged actions are supported and execute if any staged action fails.

## Template Variables in Staged Actions

When an action runs with "Execute before file publish", the template variables reflect both the temporary file (for access) and the original filename (for logical use):

- `{{.VirtualPath}}` and `{{.FsPath}}` point to the **temporary file** — use these when the action needs to access or read the file content.
- `{{.ObjectName}}` is the **original filename** as uploaded by the client (e.g., `report.pdf`) — use this when you need the logical file name (e.g., in notification messages, computing new names).
- The **published path** — where the file will appear once the actions succeed — is the directory of `{{.VirtualPath}}` joined with `{{.ObjectName}}`: `{{pathJoin (stringSlice (pathDir .VirtualPath) .ObjectName)}}`. Use it in notifications (including failure actions) and as the base for derived output names, for example `{{pathJoin (stringSlice (pathDir .VirtualPath) .ObjectName)}}.pgp` as the PGP encrypt target. See [path manipulation helpers](placeholders.md#path-manipulation).

The same applies to the ICAP quarantine path. A quarantine path ending with `/` is a directory, created when missing, and the file is placed inside it under its current name, the temporary one. A path without the trailing `/` is the full name of the quarantined file: `/quarantine/{{.Name}}/{{pathJoin (stringSlice (pathDir .VirtualPath) .ObjectName)}}` keeps the original name and directory layout, partitioned by user. See [ICAP](filesystem-actions.md#icap) for the quarantine settings.

After the staged action completes and the file is renamed, any post-upload actions in separate rules receive all variables pointing to the final path.

## Limitations

- **Requires atomic upload mode.** If atomic uploads are not enabled, the "Execute before file publish" option is ignored: actions with only this option are skipped and a warning is logged; actions that also have "Synchronous execution" run as regular synchronous actions on the file at its final path, so the file is visible while they run.
- **SFTPFs with buffering.** Remote SFTP backends with `buffer_size > 0` do not support atomic uploads. Staged actions are not available for this configuration.
- **No client notification on failure (async mode only).** In async mode, the client receives a success response before actions run, so there is no protocol-level way to inform the client if an action fails. Use event rules with email or HTTP actions to notify administrators. In sync mode (both options enabled), the client receives the error directly.
- **Filesystem rename/move actions are not recommended.** Actions that modify the file location (rename, move) during staged execution break the subsequent atomic rename — the temp file is no longer at the expected path. Use rename/move actions in separate post-upload rules instead, where they operate on the file at its final path.
