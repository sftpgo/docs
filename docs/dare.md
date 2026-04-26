---
description: "Enable transparent data-at-rest encryption in SFTPGo. Files are encrypted with AES-256-GCM before writing to disk and decrypted on read."
---

# Data At Rest Encryption (DARE)

SFTPGo supports transparent data-at-rest encryption via its `cryptfs` virtual filesystem. When enabled, files are automatically encrypted during upload and decrypted during download — the files stored on disk are always encrypted while clients see and work with the unencrypted content.

Data At Rest Encryption is available for the local filesystem backend. For cloud storage backends (S3, GCS, Azure Blob), use their native server-side encryption features.

## How it works

SFTPGo uses `AES-256-GCM` or `ChaCha20-Poly1305` for authenticated encryption (AES-GCM is selected automatically when the CPU has hardware support). Each file is encrypted with a unique, randomly generated key derived from the configured passphrase. The per-file key is **never stored** — it is derived on the fly from the passphrase and a random initialization vector using [HKDF](http://tools.ietf.org/html/rfc5869){:target="_blank"} (RFC 5869). The initialization vector is stored alongside the encrypted file.

The file size visible to clients (via SFTP, FTP, etc.) is the **decrypted size**, automatically computed by SFTPGo. The actual on-disk size is slightly larger due to encryption overhead.

## Configuration

| Parameter | Description |
| ----------- | ------------- |
| **Passphrase** | The master passphrase used to derive per-file encryption keys. Required. Stored encrypted according to your [KMS configuration](kms.md). If this passphrase is lost, all encrypted files become unrecoverable. |
| **Read buffer size** | Buffer size in MB for read (decryption) operations. `0` means no buffering. Increasing this value may improve read performance for large files. |
| **Write buffer size** | Buffer size in MB for write (encryption) operations. `0` means no buffering. Increasing this value may improve write performance for large files. |

:warning: When setting up an encrypted filesystem for a user, the home directory must point to an empty directory. If it contains existing unencrypted files, SFTPGo will attempt to decrypt them and fail.

## Limitations

- **Upload resume is not supported** — encryption is sequential and stream-based; random-access writes are not possible.
- **Truncate is not supported**.
- **Opening a file for both reading and writing at the same time is not supported** — clients that require advanced filesystem-like features (e.g., `sshfs`) are not supported.
