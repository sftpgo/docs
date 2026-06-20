---
title: "Access control and permissions"
description: "How access control works in SFTPGo: storage isolation with home directory and key prefix, virtual folders, per-directory permissions, file pattern filters, IP and protocol restrictions, and time-based access. Restrict what each user can see and do."
keywords: "SFTPGo access control, per-directory permissions, file pattern filters, restrict user access, IP whitelist, key prefix, chroot, ACL"
---

# Access control and permissions

SFTPGo controls what each user can see and do through several independent mechanisms. They operate at different layers: some decide *what storage is exposed at all*, others decide *what the user may do within the storage they can reach*, and others gate access by *network, protocol, or time*. Understanding which layer solves your problem avoids over-relying on a single setting.

| Layer | Mechanism | Controls |
| ----- | --------- | -------- |
| Storage | [Home directory / key prefix](#storage-isolation) | The user's root. Everything above it is unreachable. |
| Storage | [Virtual folders](#virtual-folders) | Which specific locations are mounted into the user's namespace. |
| Operations | [Per-directory permissions](#per-directory-permissions) | What the user may do (list, download, upload, …) in each directory. |
| Visibility | [File pattern filters](#file-pattern-filters) | Which file and directory names are listed and transferable. |
| Connection | [IP and protocol restrictions](#ip-and-protocol-restrictions) | Where the user may connect from and with which protocol/login method. |
| Connection | [Time-based access](#time-based-access) | When the user may connect. |

:information_source: These layers compose. A robust setup typically isolates the user at the storage layer, then refines operations with per-directory permissions, and optionally narrows the connection with IP and time restrictions. Settings can be assigned directly on the user or inherited from [groups](groups.md).

## Storage isolation

Each user is confined to a single storage root, which behaves like a chroot: the user cannot address anything above it.

- **Local filesystem:** the root is the user's **home directory**.
- **Cloud backends (Azure Blob, S3, Google Cloud Storage):** the root is defined by the **key prefix**. The user only sees objects whose name starts with the prefix, and the prefix is stripped from what they see — it becomes their `/`. With an empty prefix the whole bucket/container is exposed.
- **SFTP backend:** the root is the configured **SFTP root directory** (prefix) on the remote server. Unlike the cloud key prefix, this is a path on a real remote filesystem reached with the configured account; see [SFTP root directory and path confinement](sftpfs.md#sftp-root-directory-and-path-confinement) for what it guarantees, especially on a remote server you do not control.

For example, a single Azure Blob container holding `account/inbound/`, `account/outbound/` and `account/custom/` can be scoped to one tenant by setting the user's key prefix to `account/custom/`: that user sees only the contents of `custom`, addressed as `/`, and cannot reach or even name the sibling folders. This is the strongest way to expose only a subtree, because the rest is not part of the user's filesystem at all.

The optional **start directory** narrows the entry point within the existing root without changing the root itself, so the user lands directly in a chosen subdirectory.

## Virtual folders

When a user needs access to more than one location — or to a location on a different backend — mount each as a [virtual folder](virtual-folders.md) at a virtual path of your choice. A virtual folder is an independent storage definition (its own backend, bucket/container, key prefix, and optionally its own quota). Only what you explicitly mount is visible.

To expose a single shared directory and nothing else, give the user an otherwise minimal root and mount just that directory as a virtual folder. Locations you do not mount cannot be listed or accessed.

## Per-directory permissions

Permissions grant a set of operations on a virtual path. The available permissions are:

| Permission | Allows |
| ---------- | ------ |
| `*` | All permissions. |
| `list` | List the contents of files and directories. |
| `download` | Download files. |
| `upload` | Upload files. |
| `overwrite` | Overwrite an existing file while uploading. Requires `upload`. |
| `delete` | Delete files and directories (equivalent to `delete_files` + `delete_dirs`). |
| `delete_files` | Delete files. |
| `delete_dirs` | Delete directories. |
| `rename` | Rename files and directories (equivalent to `rename_files` + `rename_dirs`). |
| `rename_files` | Rename files. |
| `rename_dirs` | Rename directories. |
| `create_dirs` | Create directories. |
| `create_symlinks` | Create symbolic links. |
| `chmod` | Change file or directory permissions. |
| `chown` | Change file or directory owner and group. |
| `chtimes` | Change file or directory access and modification times. |
| `copy` | Copy files and directories server-side. |

The `/` path is mandatory for every user and acts as the fallback. Permissions are resolved by **longest-prefix match**: for any path, SFTPGo uses the permissions of the closest ancestor directory that has an explicit entry, falling back to `/`.

- **Inheritance** is automatic: a subdirectory with no explicit entry inherits from its nearest parent that has one.
- **Override** is per path: define an explicit entry for a subdirectory to change its rights. The entry **replaces** the inherited set for that subtree — it does not merge with the parent.
- **Precedence:** the most specific (deepest) matching path always wins.

:information_source: Operations that span two locations require permission on both. A **rename** or server-side **copy** needs the relevant permission on both the source and the destination directory, and creating a **symbolic link** needs `create_symlinks` on both the link's directory and the directory it points into.

For example, to allow a user to work only inside `/account/custom` within a tree mounted at `/account`, grant the desired permissions on `/account/custom` and give `/account/inbound` and `/account/outbound` an explicit entry with no permissions: those directories then cannot be entered, listed, or transferred.

```json
{
  "permissions": {
    "/": ["list"],
    "/account/custom": ["*"],
    "/account/inbound": [],
    "/account/outbound": []
  }
}
```

In this example the user can browse down to `/account`, has full access inside `/account/custom`, and is denied every operation in the two sibling directories. A path with no explicit entry — for instance `/account/custom/2024` — inherits the `*` granted on `/account/custom`.

:information_source: Most operations dereference a symbolic link and run under the permissions of the path the client requests, not of the link's target. In the example above SFTPGo will not create a link pointing into `/account/inbound` or `/account/outbound`, since they grant no `create_symlinks`; but a symbolic link already present on the storage (created outside SFTPGo) is still dereferenced. When relying on per-directory permissions to wall off sibling directories, keep symbolic-link creation disabled — its default [`symlink_mode`](config-file.md#symbolic-links-and-permissions) — and provision the storage layout accordingly. See [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions).

:information_source: Per-directory permissions control what a user may *do*, not whether a directory *name* is visible. In the example above, `inbound` and `outbound` still appear when listing `/account`, even though access is denied. To also hide the names, use a file pattern filter with the hide policy, or isolate at the storage layer so the siblings are never exposed.

## File pattern filters

File pattern filters restrict which **names** are listed and transferable within a directory. For a given path you define case-insensitive, shell-like allowed and denied patterns; allowed patterns are evaluated before denied ones. Like permissions, a filter defined for a path also applies to its subdirectories unless a more specific filter is defined.

Matching is performed on the file or directory **name**, and applies to both files and directories. Each filter has a deny policy:

| Deny policy | Behavior |
| ----------- | -------- |
| **Default** | A denied file or directory is still shown in directory listings but cannot be uploaded, downloaded, overwritten, or renamed. |
| **Hide** | The same restrictions apply, and the denied item is also removed from directory listings. May affect performance for very large directories. |

:warning: Because matching is name-based, file pattern filters are a visibility and transfer control, not a content filter. A rule that denies `*.exe` matches on the name only, so the same content uploaded as `*.txt` is allowed. Use file pattern filters to organize what is listed and to block transfers by name; to control which directories a user can reach, rely on storage isolation, virtual folders, or per-directory permissions.

Two rules govern how allowed and denied patterns interact:

- Allowed patterns are checked first; a name that matches one is accepted even if it would also match a denied pattern.
- A non-empty allowed list means **deny by default**: anything that does not match an allowed pattern is denied. An empty allowed list means **allow by default**: everything is accepted except names matching a denied pattern.

### Examples

**Allow only images in a directory.** On `/photos`, set allowed patterns `*.jpg`, `*.png`. Only those names can be uploaded, downloaded or listed; `report.txt` is denied because the allowed list is non-empty.

```json
{ "path": "/photos", "allowed_patterns": ["*.jpg", "*.png"] }
```

**Block executables everywhere.** On `/`, set denied patterns `*.exe`, `*.bat`, `*.sh`. Everything is allowed except those names. With the default deny policy they remain visible but cannot be transferred; renaming the same content to another extension bypasses the rule (matching is by name).

```json
{ "path": "/", "denied_patterns": ["*.exe", "*.bat", "*.sh"] }
```

**Hide sibling directories from listings.** On `/account`, deny the directory names `inbound`, `outbound` with the **hide** policy. Both vanish from the listing of `/account`; combine with per-directory permissions to also block access.

```json
{ "path": "/account", "denied_patterns": ["inbound", "outbound"], "deny_policy": 1 }
```

**Show only a chosen subtree.** On `/`, allow `public` with the **hide** policy: every top-level name except `public` is hidden, so the user effectively sees only `/public`. Because a filter is inherited by subdirectories, this also hides non-matching names deeper in the tree until a more specific filter overrides it.

```json
{ "path": "/", "allowed_patterns": ["public"], "deny_policy": 1 }
```

**Inheritance with an override.** A filter on `/` applies to the whole tree; add a more specific filter on `/incoming` to relax or tighten the rule just there. The deepest matching path wins, exactly like permissions.

```json
[
  { "path": "/", "denied_patterns": ["*.exe"], "deny_policy": 1 },
  { "path": "/incoming", "allowed_patterns": ["*"] }
]
```

## How path-based restrictions are evaluated

Per-directory permissions, [file pattern filters](#file-pattern-filters), allowed and denied [share paths](tutorials/shares.md#restricting-shareable-paths), and every other setting keyed on a path are evaluated against the **virtual path the client requests**, not against the physical location that path ultimately resolves to. The configured restrictions belong to the requested path; they do not follow a redirection in the underlying storage.

When the storage redirects one path to another — through a symbolic link, a Windows junction, a bind mount, a hard link, or any similar operating-system or configuration mechanism — SFTPGo applies the restrictions configured for the path the client named. If `/path1` resolves to `/path2`, the access decision uses what is configured for `/path1`, **even when `/path2` is configured with stricter restrictions**. A tighter configuration on `/path2` does not protect it from access through `/path1`, and the restrictions defined for `/path2` are not consulted.

The practical consequence is that these restrictions are only as strong as the storage layout beneath them. SFTPGo confines symbolic links to the user's accessible area and disables link creation by default ([`symlink_mode`](config-file.md#symbolic-links-and-permissions)), but redirections created directly on the storage, outside SFTPGo, cannot always be detected — Windows junctions and bind mounts in particular are not resolved. Provision the storage so its directory structure reflects the boundaries you intend, and do not place a link, junction, or mount inside a user's tree that points at a location you restrict elsewhere.

The [`create_symlinks` permission](#per-directory-permissions) is itself path-based and bidirectional: a directory that grants it can both host links and be the target of links created elsewhere. A user holding `create_symlinks` in two directories can link one to the other and reach either under the more permissive directory's rights. Do not grant `create_symlinks` on a directory whose operation permissions you restrict, and see [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) for the full interaction.

Provision the storage so its directory structure reflects the boundaries you intend.

## IP and protocol restrictions

You can gate the connection itself, independently of the filesystem:

- **Allowed/denied IP addresses** — restrict the source addresses a user may connect from, in CIDR notation (for example `192.0.2.0/24` or `2001:db8::/32`). Denied rules are evaluated before allowed ones.
- **Allowed/denied protocols** — limit the user to specific protocols (SFTP, FTP, WebDAV, HTTP).
- **Allowed/denied login methods** — limit which authentication methods the user may use (for example, public key only).
- **Two-factor requirement per protocol** — require an additional one-time code on the selected protocols.

## Time-based access

Access can be limited to specific weekly time windows. Each period defines a day of the week and a `from`–`to` interval in `HH:MM`; the user may connect only during one of the configured periods. This complements, rather than replaces, the filesystem and network controls above.

## Choosing the right mechanism

- The user should only ever work inside one subtree → **isolate at the storage layer** (home directory for local storage, key prefix for cloud backends), or **mount just that directory as a virtual folder**. The rest cannot be discovered.
- The user needs the shared parent but with different rights per subdirectory → **per-directory permissions**, optionally combined with the **hide** deny policy to also hide the denied names.
- You want to organize listings or block specific names from being transferred → **file pattern filters**.
- You want to restrict where, how, or when a user connects → **IP, protocol, login method**, and **time-based** restrictions.
