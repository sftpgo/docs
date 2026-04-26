---
description: "Map directories from any storage backend into SFTPGo user namespaces using virtual folders. Supports S3, Azure, GCS, SFTP, and local filesystems."
---

# Virtual Folders

Virtual folders act as flexible links to any supported storage backend, making that storage accessible to users at specific folder paths within their file system. This means you can present different storage systems—like local disks, cloud buckets, or external SFTP servers—as if they were simple folders, tailored to each user’s needs.

To illustrate, a user might have a virtual folder mapped to an Amazon S3 bucket, allowing them to interact with cloud storage seamlessly. Conversely, another user could have a virtual folder backed by a local encrypted filesystem. This flexibility lets you mix and match storage backends, providing a unified and convenient experience regardless of where the actual data resides. There is no fixed limit to the number of virtual folders that can be assigned to a single user.

Beyond simple access, virtual folders can be integrated with the [EventManager](eventmanager.md) to automate file transfers between storage backends. For example, after a file upload or on a scheduled basis, files can be automatically moved or copied from one storage backend to another, simplifying workflows and ensuring data is where it needs to be without manual intervention.

SFTPGo will try to automatically create any missing parent directory for the configured virtual folders at user login.

For each virtual folder, the following properties can be configured:

| Property | Description |
| ---------- | ------------- |
| **Folder name** | Unique identifier for the folder. |
| **Filesystem** | Storage backend: local, cloud (S3, GCS, Azure Blob), remote SFTP/FTP, encrypted, or HTTP. |
| **Virtual path** | Absolute path seen by SFTPGo users where the folder is mounted (e.g., `/shared`). |
| **Quota size** | Maximum size in bytes. `0` = unlimited, `-1` = included in the user's overall quota. |
| **Quota files** | Maximum number of files. `0` = unlimited, `-1` = included in the user's overall quota. |

For example, if a folder uses `/srv/data/shared` as its filesystem path and `/shared` as the virtual path, SFTPGo users can access `/srv/data/shared` via the `/shared` virtual path.

It is also possible to mount a virtual folder at the user's root path (`/`). This can be useful for sharing the same storage across multiple users — the user's own root filesystem is hidden in this case.

See the [Getting Started guide](initial-configuration.md#virtual-folders-and-permissions) for an introduction with screenshots.

Nested SFTP folders using the same SFTPGo instance (identified using the host keys) are not allowed as they could cause infinite SFTP loops.

The same virtual folder can be shared among multiple users, and you can set different quota limits for each user on that shared folder.
Alternatively, folder quotas can be included within a user’s overall quota. In this case, the folder is considered “private,” and sharing it with others will cause incorrect quota calculations for the users involved.

Folders that use dynamic paths (set through groups) must always be private to keep storage limits accurate.

When calculating a user’s quota, the system sums the sizes of all files in their home directory plus the files contained in each virtual folder that contributes to their quota.

For private folders, the storage limit counts only for the individual user who owns it, not for the folder itself. So sharing these folders with others can cause errors in tracking storage use.

If you create virtual folders that point to nested or overlapping paths, the quota calculations may become inaccurate. For example:

- `folder1` uses `/srv/data/mapped` or `C:\mapped` as mapped path
- `folder2` uses `/srv/data/mapped/subdir` or `C:\mapped\subdir` as mapped path

When you upload a file to folder2, only its quota will be updated, while the quota for folder1 will not reflect this change. This behavior is allowed to provide greater flexibility, but if you want to enforce accurate disk quotas in SFTPGo, it’s best to avoid using folders with nested paths.
Although this example refers to local folders, the same principle applies to cloud storage backends.

Using the REST API you can:

- monitor folders quota usage
- scan quota for folders
- inspect the relationships among users and folders
- delete a virtual folder. SFTPGo removes folders from the data provider, no files deletion will occur

If you remove a folder, from the data provider, any users relationships will be cleared up. If the deleted folder is mounted on the user's root (`/`) path, the user is still valid and its root filesystem will no longer be hidden. If the deleted folder is included inside the user quota you need to do a user quota scan to update its quota. An orphan virtual folder will not be automatically deleted since if you add it again later, then a quota scan is needed, and it could be quite expensive, anyway you can easily list the orphan folders using the REST API and delete them if they are not needed anymore.
