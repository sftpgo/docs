---
description: "Map directories from any storage backend into open-source SFTPGo user namespaces using virtual folders. Supports S3, Azure, GCS, SFTP, and local filesystems."
---

# Virtual Folders

A virtual folder is a mapping between a SFTPGo virtual path and a filesystem path outside the user home directory or on a different storage provider.

For example, you can have a local user with an S3-based virtual folder or vice versa.

SFTPGo will try to automatically create any missing parent directory for the configured virtual folders at user login.

For each virtual folder, the following properties can be configured:

- `folder_name`, is the ID for an existing folder. The folder structure contains the absolute filesystem path to map as virtual folder
- `filesystem`, this way you can map a local path or a Cloud backend to mount as virtual folders
- `virtual_path`, absolute path seen by SFTPGo users where the mapped path is accessible
- `quota_size`, maximum size allowed as bytes. 0 means unlimited, -1 included in user quota
- `quota_files`, maximum number of files allowed. 0 means unlimited, -1 included in user quota

For example if a folder is configured to use `/tmp/mapped` or `C:\mapped` as filesystem path and `/vfolder` as virtual path then SFTPGo users can access `/tmp/mapped` or `C:\mapped` via the `/vfolder` virtual path.

:information_source: For folders on the local filesystem, the mapped path is resolved like any other path, so a redirection placed on it is followed. Keep the mapped path and the directories above it under administrative control and writable only by trusted accounts, as described for [home directories](localfs.md).

Renaming between the user's home directory and a virtual folder, or between two virtual folders, is performed as a copy followed by a delete, even when both sides are on the local filesystem. The operation can be slow for large directory trees and is not atomic: the source is removed only after the copy fully succeeds, so no data is lost, but a failure midway can leave the files already transferred at the destination. Symbolic links and other non-regular files inside a renamed directory are skipped, not copied. File permissions are not preserved across such a rename and the modification time of regular files is restored on a best-effort basis.

Nested SFTP folders using the same SFTPGo instance (identified using the host keys) are not allowed as they could cause infinite SFTP loops.

The same virtual folder can be shared among users, different folder quota limits for each user are supported.
Folder quota limits can also be included inside the user quota but in this case the folder is considered "private" and sharing it with other users will break user quota calculation.
Folders with dynamic paths added via groups must be private to avoid breaking quota calculations.
The calculation of the quota for a given user is obtained as the sum of the files contained in his home directory and those within each defined virtual folder included in its quota.
For private folders, the quota is updated only for the matching user not for the folder itself.

If you define folders that point to nested paths or to the same path, the quota calculation will be incorrect. Example:

- `folder1` uses `/srv/data/mapped` or `C:\mapped` as mapped path
- `folder2` uses `/srv/data/mapped/subdir` or `C:\mapped\subdir` as mapped path

If you upload a file to `folder2` its quota will be updated but the quota of `folder1` will not. We allow this for more flexibility, but if you want to enforce disk quotas using SFTPGo, avoid folders with nested paths.

It is allowed to mount a virtual folder in the user's root path (`/`). This might be useful if you want to share the same virtual folder between different users. In this case the user's root filesystem is hidden from the virtual folder.

## Sub-path mounts

A folder mapping can re-root the mount onto a sub-path of the folder: the mount at `virtual_path` serves the folder starting from the configured `subpath`, which stays invisible in the paths the user sees. For example, a folder mapped at `/files` with subpath `/tenants/alice` presents the content of `tenants/alice` directly at `/files`.

The sub-path applies to the folder's backend path: the key prefix for S3, Google Cloud Storage and Azure Blob folders, the prefix for SFTP folders, and the filesystem path for local and encrypted folders. Access through a sub-path mount of an HTTP/S folder fails.

The value is a canonical POSIX path with a leading slash, for example `/dataset1/2026`, at most 191 characters. The same folder can be mapped multiple times with distinct subpaths, at freely chosen virtual paths: for example `/current` can serve the folder's `/2026` sub-path while `/archive` serves `/2025`. Quota remains a property of the whole folder: every mapping of the same folder must use the same quota limits and usage is tracked once across all its mounts.

On group mappings the subpath supports the `%username%` and `%role%` placeholders, resolved per user when the group settings are applied: a single group mapping with subpath `/tenants/%username%` gives each member their own sub-tree of a shared folder, at the same virtual path for every user. A mapping whose placeholder has no value for a user is skipped for that user.

Using the REST API you can:

- monitor folders quota usage
- scan quota for folders
- inspect the relationships among users and folders
- delete a virtual folder. SFTPGo removes folders from the data provider, no files deletion will occur

If you remove a folder, from the data provider, any users relationships will be cleared up. If the deleted folder is mounted on the user's root (`/`) path, the user is still valid and its root filesystem will no longer be hidden. If the deleted folder is included inside the user quota you need to do a user quota scan to update its quota. An orphan virtual folder will not be automatically deleted since if you add it again later, then a quota scan is needed, and it could be quite expensive, anyway you can easily list the orphan folders using the REST API and delete them if they are not needed anymore.
