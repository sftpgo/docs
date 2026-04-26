---
description: "Use Azure Blob Storage as an SFTPGo backend with shared key, SAS token, or default Azure credential (Managed Identity) authentication."
---

# Azure Blob Storage backend

SFTPGo can use Azure Blob Storage as a storage backend. The behavior and limitations are similar to the [S3 backend](s3.md).

## Authentication

SFTPGo supports three authentication methods:

1. **Shared Key** — Provide an account name and account key. The container name is required.
2. **Shared Access Signature (SAS)** — Provide a SAS URL. If the URL includes the container name, the container field is optional; otherwise, it must be specified separately.
3. **Default Azure Credentials** — Leave both the account key and SAS URL blank. SFTPGo will use Azure's default credential chain (environment variables, managed identity, Azure CLI, etc.). This is the recommended approach for deployments on Azure.

The account key and SAS URL are stored encrypted according to your [KMS configuration](kms.md).

## Configuration

| Parameter | Description |
| ----------- | ------------- |
| **Container** | Azure Blob Storage container name. Must already exist. Optional if embedded in the SAS URL. |
| **Account name** | Azure storage account name. Required when using shared key authentication. |
| **Account key** | Storage account key. Leave blank to use SAS URL or default credentials. |
| **SAS URL** | Shared Access Signature URL. Leave blank to use shared key or default credentials. |
| **Endpoint** | Azure Blob Storage endpoint. Default: `blob.core.windows.net`. Set a custom endpoint for emulators like [Azurite](https://github.com/Azure/Azurite){:target="_blank"} (e.g., `http://127.0.0.1:10000`). |
| **Key prefix** | Optional. Restricts the user to a "folder" within the container. Each user can only access objects under their assigned prefix. The prefix does not need to exist beforehand. |
| **Access tier** | Blob [access tier](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview){:target="_blank"} for uploaded objects: `Hot`, `Cool`, or `Archive`. Leave blank for the container's default tier. |
| **Skip TLS verify** | Accept any TLS certificate. :warning: Only for testing. |

### Multipart upload and download tuning

SFTPGo uses multipart operations for transferring files to/from Azure Blob Storage.

| Parameter | Default | Range | Description |
| ----------- | --------- | ------- | ------------- |
| **Upload part size** | 5 MB | 1–2000 MB | Size of each block in a block upload. |
| **Upload concurrency** | 5 | 0–64 | Number of blocks uploaded in parallel. |
| **Download part size** | 5 MB | 1–2000 MB | Size of each part in a parallel download. |
| **Download concurrency** | 5 | 0–64 | Number of parts downloaded in parallel. |

:information_source: If the upload bandwidth between the client and SFTPGo is greater than the bandwidth between SFTPGo and Azure, the client may need to wait for the final blocks to be uploaded, potentially causing a timeout. Adjust part size and concurrency accordingly.

### Checksum verification

Every upload sends a CRC64 transactional checksum (`x-ms-content-crc64` header) that Azure verifies server-side on each `Put Blob` and `Put Block` request. The checksum is precomputed over the in-memory buffer, so no extra data is buffered or re-read.

This is enabled by default and can be disabled by setting the `SFTPGO_HOOK__AZBLOB__DISABLE_CHECKSUM` environment variable to `1`. Useful for emulators or gateways that do not accept the `x-ms-content-crc64` header.

## Limitations

This backend has the same limitations as the [S3 backend](s3.md):

- `chown`, `chmod`, `truncate`, `symlink`, and `readlink` are not supported.
- Opening a file for both reading and writing at the same time is not supported.
- `rename` is implemented as server-side copy + delete — not atomic.
- Renaming non-empty directories is not supported.
- Upload resume is disabled by default and requires re-uploading the entire file when enabled.
- A local home directory is required for temporary files, unless in-memory pipes are enabled via `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1`.
- Clients that require advanced filesystem-like features (e.g., `sshfs`) are not supported.

Unlike S3, Azure Blob Storage supports setting modification times via object metadata.
