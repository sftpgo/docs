---
description: "Configure the local filesystem storage backend in SFTPGo with per-user home directories, cross-device rename support, and file path validation."
---

# Local filesystem

The local filesystem is the default storage backend. Each user is restricted to their **home directory** — all file operations (uploads, downloads, listings) are confined within it.

## Home directory

The home directory is set per-user and represents the root of their visible filesystem. It must be an absolute path (e.g., `/srv/sftpgo/data/alice`). If `users_base_dir` is configured in the data provider settings, new users will automatically get a home directory derived from `users_base_dir/username`.

SFTPGo does not create the home directory automatically. You can use an Event Manager rule on the `add` provider event to create it, or ensure it exists before the user's first login.

## Symbolic links

Symbolic links that point outside the user's home directory are **not followed** — this is enforced for security. If you need to give a user access to a directory outside their home, use a [virtual folder](virtual-folders.md) instead.

## Permissions

File and directory permissions on the underlying filesystem apply as expected. SFTPGo runs under a system user (typically `sftpgo` on Linux packages, `LOCAL SYSTEM` on Windows), so the home directory and every parent directory must be accessible to that account.

### A common source of "permission denied" on upload

On Linux, if you configure a home directory outside the packaged `users_base_dir` (which defaults to `/srv/sftpgo/data`) and the path was created as `root`, the `sftpgo` service account cannot write to it — the user authenticates but every `MKDIR` / `PUT` fails. Fix by transferring ownership and making sure every parent is traversable:

```shell
sudo mkdir -p /data/sftpgo/alice
sudo chown -R sftpgo:sftpgo /data/sftpgo
sudo chmod 755 /data
```

The packaged `/srv/sftpgo` tree is already chowned by the post-install script, so users whose home sits under it work out of the box.

On Windows, grant the service account read / write on the chosen path (via the directory properties or `icacls`) if you changed the service identity away from `LOCAL SYSTEM`.

The `setstat_mode` [configuration parameter](config-file.md) controls how SFTPGo handles requests to change file attributes (permissions, ownership, timestamps):

- `0` (default) — all requests are executed on the underlying filesystem.
- `1` — all attribute change requests are silently ignored.
- `2` — requests are silently ignored only for operations not supported by the current storage backend.

This setting is particularly relevant for **modification times** (`chtimes`). On the local filesystem, modification times are fully supported. However, some storage backends (notably S3) do not support setting modification times at all. If your users connect to a mix of local and cloud-backed paths, setting `setstat_mode` to `2` avoids errors on cloud backends while preserving full functionality on local storage.

## Shared storage and NFS

If you use a network filesystem (NFS, CIFS/SMB) as the backing store for user home directories, keep in mind that latency on network filesystems directly affects file transfer performance.

## Atomic uploads

When atomic uploads are enabled (`upload_mode` set to `1` or `2`), files are written to a temporary file in the same directory and renamed to the final path on completion. This is an atomic operation on local filesystems — the rename is instantaneous regardless of file size.
