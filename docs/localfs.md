---
description: "Configure the local filesystem storage backend in SFTPGo with per-user home directories, cross-device rename support, and file path validation."
---

# Local filesystem

The local filesystem is the default storage backend. Each user is restricted to their **home directory**: all file operations (uploads, downloads, listings) are confined within it.

## Home directory

The home directory is set per-user and represents the root of their visible filesystem. It must be an absolute path (e.g., `/srv/sftpgo/data/alice`). If `users_base_dir` is configured in the data provider settings, new users automatically get a home directory derived from `users_base_dir/username`.

A missing home directory is created at the user's first login, with the privileges of the account SFTPGo runs under. Create it beforehand when its parent directory is not writable by that account, or when it needs a specific owner or mode, see [Permissions](#permissions).

To give a user access to a directory outside the home directory, use a [virtual folder](virtual-folders.md). A rename between the home directory and a virtual folder is performed as a copy followed by a delete, see [Renaming across folders](virtual-folders.md#renaming-across-folders).

## Permissions

File and directory permissions on the underlying filesystem apply as expected. SFTPGo runs under a system user (typically `sftpgo` on Linux packages, `LOCAL SYSTEM` on Windows), so the home directory and every parent directory must be accessible to that account.

### A common source of "permission denied" on upload

On Linux, if you configure a home directory outside the packaged `users_base_dir` (which defaults to `/srv/sftpgo/data`) and the path was created as `root`, the `sftpgo` service account cannot write to it: the user authenticates but every `MKDIR` / `PUT` fails. Fix by transferring ownership and making sure every parent is traversable:

```shell
sudo mkdir -p /data/sftpgo/alice
sudo chown -R sftpgo:sftpgo /data/sftpgo
sudo chmod 755 /data
```

The packaged `/srv/sftpgo` tree is already chowned by the post-install script, so users whose home directory is under it work without further changes.

On Windows, grant the service account read / write on the chosen path (via the directory properties or `icacls`) if you changed the service identity away from `LOCAL SYSTEM`.

The `setstat_mode` [configuration parameter](config-file.md) controls how SFTPGo handles requests to change file attributes (permissions, ownership, timestamps):

- `0` (default): all requests are executed on the underlying filesystem.
- `1`: all attribute change requests are silently ignored.
- `2`: requests are silently ignored only for operations not supported by the current storage backend.

This setting is particularly relevant for **modification times** (`chtimes`). On the local filesystem, modification times are fully supported. However, some storage backends (notably S3) do not support setting modification times at all. If your users connect to a mix of local and cloud-backed paths, setting `setstat_mode` to `2` avoids errors on cloud backends while preserving full functionality on local storage.

## Shared storage and NFS

If you use a network filesystem (NFS, CIFS/SMB) as the backing store for user home directories, keep in mind that latency on network filesystems directly affects file transfer performance.

## Atomic uploads

When atomic uploads are enabled (`upload_mode` set to `1` or `2`), files are written to a temporary file in the same directory and renamed to the final path on completion. This is an atomic operation on local filesystems: the rename is instantaneous regardless of file size.

## Symbolic links

Symbolic links that point outside the user's home directory are rejected rather than followed. Within the home directory, clients holding the `create_symlinks` permission can create symbolic links when enabled by `symlink_mode` (disabled by default), and links are followed as long as their target resolves inside the home directory. Links are stored relative to the directory containing them; a link with an absolute target, created directly on the filesystem or through an older SFTPGo version, is not followed and cannot be read back, even when the target is inside the home directory. Because operations dereference a link to its target, a link can cross directories governed by different per-directory permissions. See [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) for how this interacts with per-directory permissions and the `symlink_mode` setting that controls link creation.

For environments with strict isolation requirements, provision each user from an empty directory that contains no symbolic links; symbolic-link creation is disabled by default. A link cannot escape the home directory, but as described above it can cross areas of the same tree governed by different per-directory permissions.

:warning: The home directory is reached by resolving its path, so the directories that lead to it decide where a session lands. Any redirection placed on that path, a symbolic link, a Windows junction, a bind mount, or any other construct the operating system resolves transparently, is followed, and the session is served from wherever it leads, with the privileges of the account SFTPGo runs under. Keep the home directory and every directory above it under administrative control and writable only by trusted accounts: provision them under a base directory reserved for them, as `users_base_dir` does, and give clients write access below their home rather than around it. This matters most where clients also hold local accounts on the host. The same applies to the mapped path of a [virtual folder](virtual-folders.md).

:information_source: Path confinement operates on path resolution: it constrains where symbolic links may lead, not what the filesystem itself presents beneath the home directory. A mount point, including a Linux bind mount or a special filesystem such as `/proc`, is traversed transparently, like the ordinary directory it appears as, and special files created directly on the filesystem (device nodes, named pipes) are accessed as such. SFTPGo does not create any of these itself; they appear when placed there on the host, and whatever is mounted or placed inside a user's home is then reachable under that home's permissions. Keep mount points, special filesystems and device files out of the provisioned tree unless you intend to expose their contents.

:information_source: On Windows, a client cannot create directory junctions (or other mount-point reparse points) through SFTPGo. During path resolution a junction created directly on the filesystem (for example with `mklink /J`) is treated like a symbolic link and, since a junction stores an absolute target, access through it is expected to fail rather than be followed. This handling comes from the underlying runtime, not from SFTPGo itself: do not rely on it as an isolation boundary and avoid creating junctions inside a user's home directory.

:information_source: Hard links are not symbolic links: a client cannot create one through SFTPGo, and because a hard link is a second name for the same file rather than a pointer to a path, there is nothing for SFTPGo to resolve or confine. A hard link created directly on the filesystem inside a user's home exposes the linked file's contents under the home's permissions, regardless of the per-directory permissions configured for the file's original location. Do not create hard links inside a user's home that reference files you restrict elsewhere.
