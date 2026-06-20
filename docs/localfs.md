---
description: "Configure the local filesystem storage backend in open-source SFTPGo with per-user home directories, cross-device rename support, and file path validation."
---

# Local filesystem

SFTPGo allow to restrict users to a specified directory on local filesystem, their "Home Dir".

To add a mapping for a directory outside the Home Dir you have to create a virtual folder, symbolic links outside the home directory are not allowed.

Within the home directory, clients holding the `create_symlinks` permission can create symbolic links when enabled by `symlink_mode` (disabled by default), and links are followed to their target. Because operations dereference a link to its target, a link can cross directories governed by different per-directory permissions. See [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) for how this interacts with per-directory permissions and the `symlink_mode` setting that controls link creation.

For environments with strict isolation requirements, provision each user from an empty directory that contains no symbolic links; symbolic-link creation is disabled by default. See the confinement note in [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) for the reasoning.

:information_source: On Windows, directory junctions (and other mount-point reparse points) are not symbolic links, and SFTPGo's path confinement does not resolve them. A client cannot create one through SFTPGo, but a junction created directly on the filesystem (for example with `mklink /J`) is not detected: the operating system follows it, so an operation can traverse it even when it points outside the home directory. Do not create a junction inside a user's home directory that points outside of it.

:information_source: Hard links are not symbolic links: a client cannot create one through SFTPGo, and because a hard link is a second name for the same file rather than a pointer to a path, there is nothing for SFTPGo to resolve or confine. A hard link created directly on the filesystem inside a user's home exposes the linked file's contents under the home's permissions, regardless of the per-directory permissions configured for the file's original location. Do not create hard links inside a user's home that reference files you restrict elsewhere.
