---
description: "Run actions on uploaded files before they become visible in SFTPGo. Use for antivirus scanning, content validation, or approval workflows."
---

# Execute Before File Publish

## Overview

SFTPGo supports running actions on uploaded files **before** they become visible to other users. This feature is designed for use cases where files must be verified, scanned, or processed before being published — such as antivirus scanning, content validation, DLP checks, or format conversion.

When enabled, uploaded files are held in a temporary staging area while the configured actions run. If all actions succeed, the file is automatically published to its final location. If any action fails, the file is removed and never becomes visible.

Two modes are available:

- **Asynchronous** ("Execute before file publish" only): the client receives a success response immediately without waiting for the actions to complete. Best for long-running actions where client timeout is a concern.
- **Synchronous** ("Execute before file publish" + "Synchronous execution"): the client waits for the actions to complete and receives success or failure. Best when the client needs immediate feedback and timeout is acceptable.

In both modes, the file remains hidden (at a temporary path) until all actions succeed.

## How It Works

### Asynchronous mode

1. The client uploads a file via SFTP, FTP, or HTTP/WebDAV.
2. The file is written to a temporary file (`.sftpgo-upload.<id>.<filename>`) in the same directory as the final destination.
3. The client receives a success response — the upload is complete from their perspective.
4. The configured actions run on the temporary file in the background (e.g., antivirus scan via ICAP).
5. If all actions pass: the file is renamed to its final path and becomes visible.
6. If any action fails: the temporary file is deleted and the file never appears.

### Synchronous mode

1. The client uploads a file via SFTP, FTP, or HTTP/WebDAV.
2. The file is written to a temporary file (`.sftpgo-upload.<id>.<filename>`) in the same directory as the final destination.
3. The configured actions run on the temporary file — the client waits.
4. If all actions pass: the file is renamed to its final path and becomes visible, the client receives a success response.
5. If any action fails: the temporary file is deleted, the client receives an error response.

## Requirements

- **Atomic uploads must be enabled.** Set the environment variable `SFTPGO_COMMON__UPLOAD_MODE=1`.
- **File pattern filter recommended.** Configure a deny pattern for `.sftpgo-upload*` to prevent users from accessing temporary files during processing.

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
  - **Hide (1)** — same restrictions as the default policy, plus denied files are hidden from directory listings entirely. This is the recommended option for temporary upload files so they never appear to the client.

For the typical setup, add one entry with:

| Field | Value |
| ---------- | ------- |
| Path | `/` |
| Allowed patterns | *(leave empty)* |
| Denied patterns | `.sftpgo-upload*` |
| Deny policy | Hide |

The filter applies to all directories where uploads can occur, so you do not need to add a separate entry per directory.

:information_source: These patterns use shell-style globbing (`*`, `?`, character classes), not regular expressions. The leading dot in `.sftpgo-upload*` matches the exact prefix used by SFTPGo for staging files.

### Step 3: Create an Event Action

Create an action that will process the uploaded file. Supported action types include:

- **ICAP** — Built-in antivirus/DLP scanning. Works transparently on all storage backends (local, encrypted, S3, Azure, GCS, SFTP). This is the recommended option for antivirus scanning.
- **HTTP** — Call an external webhook for custom validation.
- **Command** — Run a local script or program.
- **Any other action type** — Email notifications, filesystem operations, etc.

### Step 4: Create an Event Rule

Create a rule with:

- **Trigger:** Filesystem events
- **Events:** Upload
- **Actions:** Select your action and enable the **"Execute before file publish"** option

You can enable **both** "Execute before file publish" and "Synchronous execution" on the same action. This gives you:

- **Consumer protection:** the file is not visible during processing (stays at temp path)
- **Synchronous feedback:** the client waits for the action to complete and receives success or failure

The trade-off is potential client timeout for large files — the client blocks until the action completes. Without "Synchronous execution", the client receives an immediate success response and the action runs in the background.

## Using the Built-in ICAP Action

The ICAP action is the recommended approach for antivirus scanning and DLP checks. It:

- Accesses files through the storage abstraction layer, working on all backends
- Handles encrypted filesystems (CryptFs) transparently — files are decrypted for scanning
- Supports configurable failure policies: delete infected files, quarantine them (via virtual folders, potentially on a different storage backend), or take no action
- Works with any ICAP-compatible server

## Storage Backend Support

| Backend | Supported | Notes |
| --------- | ----------- | ------- |
| Local filesystem | Yes | Files staged in the same directory |
| Encrypted filesystem (CryptFs) | Yes | ICAP scans decrypted content transparently |
| AWS S3 | Yes | Rename uses server-side copy + delete |
| Azure Blob Storage | Yes | Rename uses server-side copy + delete |
| Google Cloud Storage | Yes | Rename uses server-side copy + delete |
| SFTP (remote) | Yes | When buffer size is 0 (direct streaming) |
| FTP (remote) | Yes | Rename via RNFR/RNTO |

:information_source: Renaming on cloud storage involves a server-side copy followed by a delete. For very large files, this adds latency proportional to the file size. This overhead is in addition to the action processing time.

## Client Behavior

### Async mode (Execute before file publish only)

- **Upload response:** The client receives a success response immediately after the upload completes. The client does not wait for actions to finish.
- **STAT after upload:** If the client issues a STAT on the file path after uploading, SFTPGo returns the file's expected size and metadata even while actions are still running. This ensures compatibility with clients that verify uploads via STAT.
- **File visibility:** The file at its final name does not appear in directory listings until all actions complete successfully. The temporary file (`.sftpgo-upload.*`) resides in the same directory: with a "Hide" deny policy it is completely invisible; with a "Deny" policy it appears in listings but all operations on it (download, delete, rename, etc.) are blocked. In both cases, the file content is inaccessible during processing.
- **Failed actions:** If an action fails, the file is silently removed. The client has already received a success response — there is no way to notify the client of the failure through the protocol. Use email or webhook actions to alert administrators.

### Sync mode (Execute before file publish + Synchronous execution)

- **Upload response:** The client blocks until all staged actions complete. If all succeed, the file is renamed and the client receives a success response. If any fails, the temp file is deleted and the client receives an error.
- **File visibility:** Same as async — the file is not visible during processing.
- **Timeout risk:** For large files (e.g., 2 GB antivirus scan taking 90-120 seconds), the client may timeout. Configure client-side timeouts accordingly.

### Operations on the file during processing (both modes)

While actions are running and the file has not been renamed yet:

- **STAT on the final path:** succeeds — SFTPGo returns the expected file size and metadata from an internal lookup, ensuring compatibility with clients that verify uploads.
- **Download, rename, delete on the final path:** fail with "file not found" — the file does not exist at the final path until the rename completes.
- **Any operation on the temp file (`.sftpgo-upload.*`):** blocked by the file pattern filter if configured (either denied or hidden depending on the policy).

## Interaction between staged and non-staged rules

When multiple rules match the same upload event, some with "Execute before file publish" and some without, the behavior is:

1. Rules with "Execute before file publish" execute **first**, on the temporary file (before rename).
2. If all staged actions succeed, the file is renamed to its final path.
3. Rules **without** "Execute before file publish" (standard sync or async actions in separate rules) execute **after** the rename, on the file at its final path — the file is visible at this point.
4. If any staged action fails, the file is never renamed — standard rules for this upload do not execute.

:warning: Standard sync actions in separate rules do **not** delay the uploader's response when staged actions are present. In async mode, the client receives OK immediately; in sync mode, only the staged actions block the client. In both cases, standard actions in other rules run after the rename, in the background. This is a known behavioral change that cannot be prevented by validation, since rules are independent and SFTPGo cannot know at rule creation time whether another rule with staged actions will match the same event. If you need an action to block the client AND run before the file is visible, use "Execute before file publish" combined with "Synchronous execution" on the same action in the same rule.

## Combining with Other Actions

You can use "Execute before file publish" alongside standard post-upload actions by placing them in **separate rules**:

- **Separate rules:** Actions in other rules (synchronous or asynchronous) execute **after** the file is published (renamed to its final path). The file is visible during the execution of these actions. If the staged action fails and the file is never published, actions in other rules do not execute for that upload.
- **Same rule:** A rule with "Execute before file publish" actions can only contain other staged actions and failure actions. Non-staged actions (webhooks, email notifications, etc.) must be placed in a separate rule. This is enforced by validation.
- **Failure actions:** Failure actions in the same rule as staged actions are supported and execute if any staged action fails.

## Template Variables in Staged Actions

When an action runs with "Execute before file publish", the template variables reflect both the temporary file (for access) and the original filename (for logical use):

- `{{.VirtualPath}}` and `{{.FsPath}}` point to the **temporary file** — use these when the action needs to access or read the file content.
- `{{.ObjectName}}` is the **original filename** as uploaded by the client (e.g., `report.pdf`) — use this when you need the logical file name (e.g., in notification messages, computing new names).

After the staged action completes and the file is renamed, any post-upload actions in separate rules receive all variables pointing to the final path.

## Limitations

- **Requires atomic upload mode.** If atomic uploads are not enabled, the "Execute before file publish" option is silently ignored and the standard upload flow applies.
- **TempPath configuration on local/encrypted filesystems.** If the server-wide `TempPath` option is set, staged actions are skipped for local and encrypted filesystems. When `TempPath` is configured, temporary upload files are placed in that directory instead of the user's home directory. Actions that access the file (such as ICAP or filesystem checks) operate through the user's virtual filesystem, which can only reach files inside the user's home directory. Since the temporary file is outside that scope, these actions cannot find or read it. Additionally, if `TempPath` and the user's home directory are on different filesystems, the final rename becomes a copy+delete operation instead of an atomic rename, adding significant latency for large files. If you need staged actions, configure file pattern filters (deny `.sftpgo-upload*`) instead of `TempPath` to hide temporary files — this achieves a similar goal while keeping files within the user's home directory where actions can access them and the rename remains atomic. Other backends (S3, Azure, GCS, SFTP, FTP) are not affected because they ignore `TempPath` and always place temporary files alongside the final destination.
- **SFTPFs with buffering.** Remote SFTP backends with `buffer_size > 0` do not support atomic uploads. Staged actions are not available for this configuration.
- **No client notification on failure (async mode only).** In async mode, the client receives a success response before actions run, so there is no protocol-level way to inform the client if an action fails. Use event rules with email or HTTP actions to notify administrators. In sync mode (both options enabled), the client receives the error directly.
- **Filesystem rename/move actions are not recommended.** Actions that modify the file location (rename, move) during staged execution break the subsequent atomic rename — the temp file is no longer at the expected path. Use rename/move actions in separate post-upload rules instead, where they operate on the file at its final path.
