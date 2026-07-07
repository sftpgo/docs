---
description: "Configure the local filesystem storage backend in open-source SFTPGo with per-user home directories, cross-device rename support, and path confinement."
---

# Local filesystem

SFTPGo allow to restrict users to a specified directory on local filesystem, their "Home Dir".

To add a mapping for a directory outside the Home Dir you have to create a virtual folder, symbolic links outside the home directory are not allowed. Renaming between the home directory and a virtual folder, or between two virtual folders, is performed as a copy followed by a delete even when both are on local storage, see [Virtual folders](virtual-folders.md) for the details.

Within the home directory, clients holding the `create_symlinks` permission can create symbolic links when enabled by `symlink_mode` (disabled by default), and links are followed as long as their target resolves inside the home directory. Links are stored relative to the directory containing them; a link with an absolute target, created directly on the filesystem or through an older SFTPGo version, is not followed and cannot be read back, even when the target is inside the home directory. Because operations dereference a link to its target, a link can cross directories governed by different per-directory permissions. See [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) for how this interacts with per-directory permissions and the `symlink_mode` setting that controls link creation.

For environments with strict isolation requirements, provision each user from an empty directory that contains no symbolic links; symbolic-link creation is disabled by default. A link cannot escape the home directory, but as described above it can cross areas of the same tree governed by different per-directory permissions.

:information_source: Path confinement operates on path resolution: it constrains where symbolic links may lead, not what the filesystem itself presents beneath the home directory. A mount point — including a Linux bind mount or a special filesystem such as `/proc` — is traversed transparently, like the ordinary directory it appears as, and special files created directly on the filesystem (device nodes, named pipes) are accessed as such. A client cannot create any of these through SFTPGo; they only appear when placed there with administrative privileges on the host, and whatever is mounted or placed inside a user's home is then reachable under that home's permissions. Keep mount points, special filesystems and device files out of the provisioned tree unless you intend to expose their contents.

:information_source: On Windows, a client cannot create directory junctions (or other mount-point reparse points) through SFTPGo. During path resolution a junction created directly on the filesystem (for example with `mklink /J`) is treated like a symbolic link and, since a junction stores an absolute target, access through it is expected to fail rather than be followed. This handling comes from the underlying runtime, not from SFTPGo itself: do not rely on it as an isolation boundary and avoid creating junctions inside a user's home directory.

:information_source: Hard links are not symbolic links: a client cannot create one through SFTPGo, and because a hard link is a second name for the same file rather than a pointer to a path, there is nothing for SFTPGo to resolve or confine. A hard link created directly on the filesystem inside a user's home exposes the linked file's contents under the home's permissions, regardless of the per-directory permissions configured for the file's original location. Do not create hard links inside a user's home that reference files you restrict elsewhere.
