---
description: "Map directories from any storage backend into SFTPGo user namespaces using virtual folders. Supports S3, Azure, GCS, SFTP, and local filesystems."
---

# Virtual Folders

A virtual folder maps a storage backend to a path of the user's virtual file system. Local disks, cloud buckets and remote SFTP servers are all presented to the user as ordinary directories.

For example, one user can have a virtual folder mapped to an Amazon S3 bucket, while another has a virtual folder backed by a local encrypted filesystem. Different backends can be combined within the same user's file system, and there is no fixed limit to the number of virtual folders assigned to a single user.

Virtual folders can also be the source or the target of [EventManager](eventmanager.md) actions, to move or copy files between storage backends after an upload or on a schedule.

See the [Getting Started guide](initial-configuration.md#virtual-folders-and-permissions) for an introduction with screenshots.

## Folders and mappings

A virtual folder is defined once, in the **Folders** section of the WebAdmin, and then mapped to one or more users or groups. The folder carries the storage, the mapping decides where and how the folder appears to the account.

A folder has the following properties:

| Property | Description |
| ---------- | ------------- |
| **Name** | Unique identifier of the folder. |
| **Description** | Optional free text. |
| **Role** | Assigns the folder to a [role](roles.md#assigning-the-role-to-groups-and-folders). The field is shown to global administrators; the folders a role administrator creates take its role. |
| **Storage** | The backend and its settings: the directory to serve for a local or encrypted disk, the bucket and key prefix for S3, GCS and Azure Blob, the server and root path for SFTP, FTP and HTTP. |

A mapping, on the user or group page, has the following properties:

| Property | Description |
| ---------- | ------------- |
| **Mount path** | Absolute path where the folder appears in the user's file system, for example `/shared`. `virtual_path` in the REST API. |
| **Quota size** | Maximum size, `0` for no limit, `-1` to count the folder within the user's quota. |
| **Quota files** | Maximum number of files, `0` for no limit, `-1` to count the folder within the user's quota. |
| **Sub-path** | Serves the folder starting from a sub-path instead of its root. See [Sub-path mounts](#sub-path-mounts). |
| **Exposed sub-paths** | Exposes the listed sub-paths of the folder, each as a directory under the mount path. See [Sub-path mounts](#sub-path-mounts). |

For example, a folder that serves the local directory `/srv/data/shared`, mapped at `/shared`, lets the user work on `/srv/data/shared` through the `/shared` path.

At login SFTPGo creates the missing parent directories of each mount path in the user's root filesystem and, for folders on local disk, the mapped directory itself. The mount path and the directories above it cannot be renamed or removed by the user while the mapping exists.

A folder can also be mounted at the user's root path (`/`) to give several users the same storage. The user's own root filesystem is hidden in this case.

## Sub-path mounts

A mapping can serve a part of the folder instead of its root, in two forms:

- **Sub-path**: the mount serves the folder starting from the given sub-path, which does not appear in the paths the user sees.
- **Exposed sub-paths**: the mount exposes the listed sub-paths, each as a directory under the mount path.

Both take absolute POSIX paths, for example `/dataset1`, and a mapping uses one form or the other. The sub-path is appended to the storage location of the folder: the key prefix for S3, Google Cloud Storage and Azure Blob folders, the root path for SFTP folders, the directory for local and encrypted folders. On [FTP](ftpfs.md#remote-directory) and [HTTP](httpfs.md#remote-directory) folders it becomes part of the remote directory and follows its rules: the backend must be listed in the [`allow_remote_directory`](config-file.md) setting and the resulting path is a starting path rather than a boundary.

### Sub-path

A folder mapped at `/files` with sub-path `/tenants/alice` presents the content of `tenants/alice` directly at `/files`. The same folder can be mapped several times with different sub-paths, at mount paths of your choice: `/current` can serve the folder's `/2026` sub-path while `/archive` serves `/2025`.

On group mappings the sub-path accepts the [placeholders](groups.md#placeholders): a single mapping with sub-path `/tenants/%username%` gives each member its own sub-tree of one shared folder, at the same mount path for every user. A member the placeholder has no value for gets no mount for that mapping and logs in with the others, as described on the groups page.

### Exposed sub-paths

A folder mapped at `/data` with exposed sub-paths `/dataset1` and `/dataset2` presents those two directories under `/data`, each serving the matching sub-path of the folder. Each entry works as a virtual folder of its own, mounted at the combined path: per-directory permissions and file pattern filters apply to that path. Multi-segment entries such as `/dataset1/jan-15` are supported, and the intermediate directories are shown while browsing to them. An entry inside another one adds a path to content the outer entry already serves.

This form fits mappings with many entries: thousands of sub-paths are a single mapping on the account. In the WebAdmin the entries are edited one per line; lists of several hundred entries are shown as a count and are managed through the REST API.

The mount path and the intermediate directories of multi-segment entries are ordinary directories of the user's root filesystem: SFTPGo creates them at login, listings show their own content together with the exposed sub-paths, and files written there are stored in the user's own filesystem. [Per-directory permissions](access-control.md#per-directory-permissions) can keep the mount path browsable while directing writes to the exposed sub-paths: for example `/data` with list and download permissions and `/data/*` with full permissions. With multi-segment entries the intermediate directories match the wildcard at their own depth.

On group mappings the entries accept the same [placeholders](groups.md#placeholders) as the sub-path: an entry a member has no value for is left out of that member's list, the others are mounted.

## Sharing a folder between users

The same folder can be mapped to several users, and each mapping can set its own quota limits: the limit of a mapping is compared with the total usage of the folder. Within a single user or group every mapping of the same folder carries the same limits, since quota belongs to the folder and covers all of its mounts.

A folder whose quota is counted within the user's quota (`-1` on both limits) is private to that user: mapping it to other users makes the quota of the users involved inaccurate.

Folders whose storage location contains a group [placeholder](groups.md#placeholders), in the mapped path or in the key prefix, give each member a different location under the same folder name. Keep them private, since a single usage counter cannot describe them and a quota scan is not available for them. A group mapping that separates members through a placeholder in the sub-path keeps one shared folder instead, with a folder-wide quota, and behaves like any other shared folder.

## Quota

The [quota of a user](users.md#quota-and-limits) is the size of the files in the user's root filesystem plus the files of every folder counted within the user's quota. The quota of a folder covers the whole folder, across all of its mounts and sub-paths, including content outside the mounted sub-paths.

Folders whose mapped paths are nested or overlapping make the counters inaccurate. For example, with `folder1` serving `/srv/data/mapped` and `folder2` serving `/srv/data/mapped/subdir`, a file uploaded to `folder2` updates its quota while the quota of `folder1` does not reflect it. The same happens when a folder's mapped path is inside the root directory of a user who mounts it: the content is reachable both at the mount path and at the matching path under the root, each governed by the permissions and file pattern filters defined for it, and operations on the root path do not update the folder's quota. The principle applies to cloud backends as well, with overlapping key prefixes. Keep mapped paths distinct and outside the users' root directories to have accurate quotas and one set of rules per folder.

## Renaming across folders

A rename between the user's root filesystem and a virtual folder, or between two virtual folders, is accepted when both sides are on the same storage: the local filesystem, the same bucket or container, the same SFTP server. Between different storages, copy the files and remove the source.

On the local filesystem such a rename is performed as a copy followed by a delete. It can be slow for large directory trees and is not atomic: each file leaves the source only after its copy succeeds, so no data is lost, but a failure midway leaves the files already moved at the destination and the rest at the source. Symbolic links and other special files are not moved: one found inside the renamed tree stays at the source, together with the source directory, and renaming such a file on its own fails. Regular files keep their permission bits, except `setuid`, `setgid` and the sticky bit, and their modification time.

## Limitations

Nested SFTP folders pointing to the same SFTPGo instance (identified by its host keys) are not allowed, as they could cause infinite SFTP loops.

For folders on local disk the mapped path is served like a root directory: see [symbolic links](localfs.md#symbolic-links) for how redirections placed on the storage are handled.

## Managing folders with the REST API

Using the REST API you can:

- monitor folders quota usage
- scan quota for folders
- inspect the relationships among users and folders
- delete a virtual folder. SFTPGo removes the folder from the data provider, no files are deleted

If you remove a folder from the data provider, any user relationships are cleared. If the deleted folder was mounted on the user's root (`/`) path, the user remains valid and its root filesystem is no longer hidden. If the deleted folder was counted within the user quota, run a user quota scan to update the quota. An orphan virtual folder is not deleted automatically, because adding it again later would require a quota scan, which can be expensive. You can list the orphan folders using the REST API and delete them if they are no longer needed.
