---
description: "SFTPGo filesystem actions: copy, rename, delete, compress, extract, PGP encrypt/decrypt, ICAP scan, and IMAP email ingestion."
---

# Filesystem Actions

Filesystem actions allow [Event Manager](eventmanager.md) rules to manipulate files and directories as part of automated workflows. SFTPGo grants the required permissions automatically — the behavior is equivalent to performing the same operations from an SFTP client, with the same underlying restrictions.

All filesystem actions support [placeholders](placeholders.md) in their path fields, making them fully dynamic. For example, you can use `{{.VirtualPath}}` to reference the file that triggered the event, or `{{.ObjectName}}` for just the filename.

## Virtual folders

Filesystem actions can operate across storage backends and outside a user's security context by using **virtual folders**:

- **Source folder** — Overrides the filesystem used to read source files. Essential for scheduled tasks or workflows where no user context is available.
- **Target folder** — Overrides the filesystem for target paths, enabling cross-backend operations (e.g., copy to a different S3 bucket, archive to an external SFTP server).

When **both** folders are specified, the action runs **once** as a system action, regardless of matching users. When only one folder is specified, the action runs per-user.

See the [Virtual Folders tutorial](tutorials/eventmanager-folders.md) for practical examples of virtual folder workflows.

## Rename

Renames one or more files or directories. Each entry defines a source path and a target path. Both support placeholders.

This is an atomic operation on local filesystems. On cloud storage backends, rename is implemented as a server-side copy followed by a delete.

## Delete

Deletes one or more files or directories.

### Deleting directory contents

Use a **trailing slash** on the path to delete the *contents* of a directory without removing the directory itself. For example:

- `/inbox/` — deletes everything inside `/inbox/` but keeps the directory.
- `/inbox` — deletes the `/inbox` directory and everything in it.

This is useful for cleanup actions that need to empty a directory periodically while preserving the directory structure.

## Create directories

Creates one or more directories, including intermediate subdirectories (equivalent to `mkdir -p`). Useful as a post-creation action to set up a standard folder structure for new users.

## Path exists

Checks whether a specified path exists. If the path does not exist, the action fails — you can use this with **Stop on failure** to conditionally skip subsequent actions.

## Copy

Copies one or more files or directories. Each copy entry defines a source path and a target path, both supporting placeholders. Copy is the most feature-rich filesystem action, with several options to fine-tune its behavior.

### Glob patterns

The source path can use **wildcards in the last path component** to select multiple files:

- `/inbox/*.csv` — copies all `.csv` files from `/inbox/`.
- `/data/report_??.txt` — copies files matching the pattern (e.g., `report_01.txt`, `report_12.txt`).

Glob patterns follow standard shell syntax (`*` matches any sequence of characters, `?` matches a single character). Subdirectories are **not** recursed into. To copy all contents of a directory, use a trailing `/` instead (e.g., `/inbox/`).

### Source disposition (After copy)

Controls what happens to the source file after a successful copy:

| Option | Behavior |
| -------- | ---------- |
| **Keep source** (default) | The source file is left unchanged. |
| **Delete source** | The source file is removed after a successful copy. |
| **Move source** | The source file is moved to a specified archive path. If the archive is on the same filesystem, an atomic rename is used; otherwise, SFTPGo falls back to a streaming copy + delete, allowing cross-backend archival (e.g., move the source from local disk to an SFTP-backed virtual folder). The move target path supports placeholders and is resolved relative to the source filesystem. |

:information_source: Source disposition only applies to files. Source directories are never removed or moved — only the files within them are disposed of.

### Max retries

Number of retry attempts per file, from 0 to 5. Each retry waits 5 seconds. Retries apply to both the file copy and the source disposition step. With `MaxRetries = 5`, a single failing file could block the pipeline for up to 25 seconds.

### Continue on error

When enabled, the action processes **all** files even if some fail. Errors are collected (up to 100 entries) and reported at the end as a combined error. Without this option, the first failure stops the entire action.

This is particularly useful when copying multiple files via glob patterns — one inaccessible file won't prevent the rest from being copied.

## Compress

Compresses one or more files and directories into a **ZIP** archive. The source paths support placeholders. Useful for bundling files before transfer or archival.

## Extract

Extracts **ZIP** archives. To mitigate common ZIP-based attacks, the following security limits are enforced by default:

| Limit | Default | Description |
| ------- | --------- | ------------- |
| Maximum compression ratio | 60 | Prevents zip bombs with extreme compression ratios. |
| Maximum number of files | 1000 | Limits the number of entries extracted from a single archive. |
| Maximum uncompressed size | 1 GB | Caps the total size of extracted content. |

These limits can be adjusted via [environment variables](env-vars.md).

## PGP

Encrypts or decrypts files using PGP-compatible encryption. Supports both password-based encryption and PGP key pairs, with optional digital signatures for authenticity and integrity verification.

When using a key pair:

- **Encryption**: the public key is required. If a private key is also provided, it is used to **sign** the encrypted output.
- **Decryption**: the private key is required. If a public key is also provided, it is used to **verify** the signature.

Source and target paths support placeholders. A common pattern is to encrypt an uploaded file and store it with a `.pgp` extension:

- Source: `/{{.VirtualPath}}`
- Target: `/{{.VirtualPath}}.pgp`

See the [PGP tutorial](tutorials/eventmanager-pgp.md) for step-by-step encryption and decryption examples.

## Metadata Check

Verifies whether a specified metadata key exists on a cloud storage object and matches a configured value. To check for the **absence** of a key, leave the value field empty.

If the condition is not met, the check is **retried** until the specified timeout is reached (when greater than zero). This is useful for waiting on asynchronous processing pipelines that set metadata upon completion.

:information_source: This action is supported only for cloud storage backends (S3, Azure Blob, GCS).

## IMAP

Integrates with IMAP mailboxes to fetch email attachments and make them available within SFTPGo. Attachments can be saved to a user's home directory or mapped into a virtual folder.

This enables automated ingestion of files delivered via email — for example, receiving invoices or reports as email attachments and making them available for download via SFTP.

When a **target folder** is specified, the action runs as a system action (once, not per-user).

## ICAP

Integrates with ICAP servers to perform **antivirus scanning** and **DLP (Data Loss Prevention)** checks on uploaded files. After a file upload, the file is streamed to an ICAP server for inspection.

ICAP accesses files through SFTPGo's storage abstraction layer, so it works transparently on **all** storage backends — local filesystem, encrypted filesystem (CryptFs), S3, Azure Blob, GCS, and remote SFTP.

### Failure policies

When an ICAP scan detects an issue, you can configure the response:

| Policy | Behavior |
| -------- | ---------- |
| **Delete** | The file is removed immediately. |
| **Quarantine** | The file is moved to a quarantine directory. By mapping the quarantine to a virtual folder, you can quarantine files to a different storage backend entirely (e.g., move infected files from the user's S3 bucket to a dedicated quarantine bucket). |
| **No action** | The scan result is logged but the file is left in place. |

### Combined with Execute Before File Publish

ICAP is the recommended action type for the [Execute Before File Publish](execute-before-file-publish.md) feature. When combined, uploaded files are scanned *before* they become visible to other users — if the scan fails, the file is deleted and never published. This provides the strongest protection against malicious uploads.
