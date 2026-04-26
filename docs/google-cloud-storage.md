---
description: "Use Google Cloud Storage as an SFTPGo backend with service account credentials, automatic authentication, and Hierarchical Namespace support."
---

# Google Cloud Storage backend

SFTPGo can use Google Cloud Storage (GCS) as a storage backend. The behavior and limitations are similar to the [S3 backend](s3.md).

## Authentication

GCS supports two authentication methods:

1. **Application Default Credentials (ADC)** — SFTPGo automatically discovers credentials from the environment (Workload Identity, service account keys, `gcloud` CLI, etc.). This is the recommended approach for deployments on Google Cloud. See the [GCP documentation](https://cloud.google.com/docs/authentication/production#providing_credentials_to_your_application){:target="_blank"} for details.
2. **JSON credentials file** — Provide a service account key file obtained from the Google Cloud Console. The credentials are stored encrypted according to your [KMS configuration](kms.md).

## Configuration

| Parameter | Description |
| ----------- | ------------- |
| **Bucket** | Required. The GCS bucket to use. Must already exist. |
| **Credentials** | Service account JSON credentials. Leave blank to use Application Default Credentials (set Automatic credentials to `1`). |
| **Automatic credentials** | Set to `1` to use Application Default Credentials instead of explicit JSON credentials. |
| **Key prefix** | Optional. Restricts the user to a "folder" within the bucket. Each user can only access objects under their assigned prefix. The prefix does not need to exist beforehand. |
| **Storage class** | GCS [storage class](https://cloud.google.com/storage/docs/storage-classes){:target="_blank"} for uploaded objects (e.g., `STANDARD`, `NEARLINE`, `COLDLINE`, `ARCHIVE`). Leave blank for the bucket's default. |
| **ACL** | Predefined [ACL](https://cloud.google.com/storage/docs/access-control/lists#predefined-acl){:target="_blank"} to apply to uploaded objects. Leave blank for the default. |
| **Hierarchical Namespace** | Set to `1` to enable Hierarchical Namespace (HNS) support for buckets with this feature enabled. See below. |
| **Universe domain** | Service domain for environments that operate outside the standard Google Cloud (`googleapis.com`), such as [Google Distributed Cloud](https://cloud.google.com/distributed-cloud){:target="_blank"} or sovereign cloud deployments with data residency requirements. Leave blank for standard Google Cloud. |
| **Skip TLS verify** | Accept any TLS certificate. :warning: Only for testing. |

### Multipart upload tuning

| Parameter | Default | Range | Description |
| ----------- | --------- | ------- | ------------- |
| **Upload part size** | 16 MB | 5–2000 MB | Size of each chunk in a multipart upload. |
| **Upload part max time** | 32 seconds | — | Maximum seconds to upload a single chunk. `0` uses the default. |

:information_source: If the upload bandwidth between the client and SFTPGo is greater than the bandwidth between SFTPGo and GCS, the client may need to wait for the upload to complete, potentially causing a timeout.

### Checksum verification

CRC32C is automatically computed by the Google Cloud Storage SDK on every upload and validated server-side by GCS. This is the canonical integrity mechanism for GCS objects — no configuration is needed.

## Hierarchical Namespace (HNS)

Standard GCS buckets use a flat namespace — "directories" are simulated via key prefixes. [Hierarchical Namespace](https://cloud.google.com/storage/docs/hns-overview){:target="_blank"} buckets provide true directory semantics with atomic folder operations.

When HNS is enabled (`Hierarchical Namespace = 1`), SFTPGo uses the Cloud Storage Control API for folder operations:

- **Atomic folder rename** — Directories are renamed as a single atomic operation instead of per-object copy + delete.
- **Explicit folder creation and deletion** — `mkdir` and `rmdir` use native folder operations.
- **Better consistency** — Folder metadata is handled directly by the storage service.

:warning: Only enable HNS if your GCS bucket has Hierarchical Namespace enabled. Using this setting with a standard (flat namespace) bucket will cause errors.

## Limitations

This backend has the same limitations as the [S3 backend](s3.md):

- `chown`, `chmod`, `truncate`, `symlink`, and `readlink` are not supported.
- Opening a file for both reading and writing at the same time is not supported.
- `rename` is implemented as copy + delete for objects. With HNS enabled, folder renames are atomic.
- Renaming non-empty directories is not supported unless HNS is enabled.
- Upload resume is disabled by default and requires re-uploading the entire file when enabled.
- A local home directory is required for temporary files, unless in-memory pipes are enabled via `SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1`.
- Clients that require advanced filesystem-like features (e.g., `sshfs`) are not supported.

Like Azure Blob Storage, GCS supports setting modification times via object metadata.
