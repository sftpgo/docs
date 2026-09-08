---
title: "Access control and permissions"
description: "How SFTPGo restricts what an account reaches: storage isolation, virtual folders, per-directory permissions, file pattern filters, IP and time rules."
keywords: "SFTPGo access control, per-directory permissions, file pattern filters, restrict user access, IP whitelist, key prefix, ACL"
---

# Access control and permissions

SFTPGo controls what each [user](users.md) can see and do through several independent mechanisms. They operate at different layers: some decide *what storage is exposed at all*, others decide *what the user may do within the storage they can reach*, and others gate access by *network, protocol, or time*. Understanding which layer solves your problem avoids over-relying on a single setting.

| Layer | Mechanism | Controls |
| ----- | --------- | -------- |
| Storage | [Home directory / key prefix](#storage-isolation) | The user's root. Everything above it is unreachable. |
| Storage | [Virtual folders](#virtual-folders) | Which additional locations are mounted into the user's file system. |
| Operations | [Per-directory permissions](#per-directory-permissions) | What the user may do (list, download, upload, ...) in each directory. |
| Visibility | [File pattern filters](#file-pattern-filters) | Which file and directory names are listed and transferable. |
| Connection | [IP and protocol restrictions](#ip-and-protocol-restrictions) | Where the user may connect from and with which protocol/login method. |
| Connection | [Time-based access](#time-based-access) | When the user may connect. |

:information_source: These layers compose. A typical setup isolates the user at the storage layer, then refines operations with per-directory permissions, and optionally narrows the connection with IP and time restrictions. Settings can be assigned directly on the user or inherited from [groups](groups.md).

## Storage isolation

The file system of a user is described in [Users](users.md#storage): a root filesystem seen as `/`, plus the virtual folders mounted into it. The root is a boundary: nothing above it is part of the user's file system, so it cannot be reached or even named. What the root is depends on the backend: the home directory on the local disk, the bucket or container narrowed by the key prefix on S3, Google Cloud Storage and Azure Blob, the configured root directory on a remote SFTP server. The SFTP root is a path on a remote filesystem reached with the configured account, see [SFTP root directory and path confinement](sftpfs.md#sftp-root-directory-and-path-confinement) for what it guarantees on a server you do not control.

For example, a single Azure Blob container holding `account/inbound/`, `account/outbound/` and `account/custom/` is scoped to one tenant by setting the user's key prefix to `account/custom/`: that user sees the contents of `custom` as `/` and the sibling folders do not exist for it. This is the strongest way to expose a single subtree.

## Virtual folders

A [virtual folder](virtual-folders.md) adds a location to the user's file system at a mount path, with a backend and a quota of its own: the root exposes one subtree, virtual folders add others. A mapping can also target a sub-path of the folder, see [Sub-path mounts](virtual-folders.md#sub-path-mounts).

## Per-directory permissions

Permissions grant a set of operations on a virtual path. The available permissions are:

| Permission | Allows |
| ---------- | ------ |
| `*` | All permissions. |
| `list` | List directory contents and read file and directory metadata. |
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
- **Override** is per path: define an explicit entry for a subdirectory to change its rights. The entry **replaces** the inherited set for that subtree, it does not merge with the parent.
- **Precedence:** the most specific (deepest) matching path always wins.

In the WebAdmin the **Permissions** field of the user holds the permissions of `/`, and the **Per-directory permissions** section holds one entry per path. Wildcards are supported in paths: `/incoming/*` matches any directory within `/incoming`.

For example, to allow a user to work only inside `/account/custom` within a tree mounted at `/account`, grant the desired permissions on `/account/custom` and give `/account/inbound` and `/account/outbound` an entry with no permissions selected:

| Path | Permissions |
| ---- | ----------- |
| `/` | `list` |
| `/account/custom` | `*` |
| `/account/inbound` | none |
| `/account/outbound` | none |

The user can browse down to `/account`, has full access inside `/account/custom`, and is denied every operation in the two sibling directories. A path with no explicit entry, for instance `/account/custom/2024`, inherits the `*` granted on `/account/custom`.

### How each operation is checked

**Reading metadata needs `list`.** It covers the attributes of a file or directory and the canonicalization of a path, not only listings. Interactive clients rely on both: the OpenSSH `sftp` client canonicalizes its working directory right after login, and `scp` on OpenSSH 9.0 and later, which transfers over SFTP, reads the attributes of the destination to learn whether it is a directory. Grant `list` on every path such clients work in; an upload-only drop box combines `list` with `upload`. Clients that write to an explicit file name, the legacy SCP protocol (`scp -O`) included, work with `upload` alone.

**Three permission pairs are selected by what the path holds.** `rename_files` or `rename_dirs` and `delete_files` or `delete_dirs` apply according to what the path holds, `upload` or `overwrite` according to whether the target name is already taken. Granting `rename` or `delete` covers both kinds.

**An operation that spans two paths is authorized on both.** A rename needs the rename permission on the parent directories of both the source and the destination; a rename onto an existing file also needs `overwrite`, and a rename onto an existing directory is refused. A server-side copy needs `copy` on both parent directories, `download` on the source and the permission to write the destination (`upload`, or `overwrite` for an existing file); copying a directory needs `copy` on the source directory and on the destination directory, and `create_dirs` where the copied directories are created. Creating a symbolic link needs `create_symlinks` on the link's directory and on the directory that contains the link's target: a link to a file is authorized by the file's directory, a link to a directory by that directory's parent.

**`copy` covers the server-side copy** offered by the REST API, the WebClient and the `sftpgo-copy` SSH command. The `pre-download` and `pre-upload` hooks are not run for it, since it is not a transfer, and the `copy` event is emitted once it completes. A WebDAV `COPY` is driven by the client through the primitives the protocol defines, so it is authorized by the read and write permissions of the paths it touches; see [WebDAV](webdav.md#known-issues-and-limitations).

:warning: **Renaming a directory moves the entries it holds, so they are authorized against both paths.** When per-directory permissions that restrict renaming apply inside either tree, or the two paths are governed by different [file pattern filters](#file-pattern-filters), the backends that move the directory in a single operation, the local filesystem included, refuse the rename; the cloud backends that move the entries individually (S3, Azure Blob and Google Cloud Storage without [Hierarchical Namespace](google-cloud-storage.md#hierarchical-namespace-hns), with [`rename_mode: 1`](config-file.md#common)) check each entry as it moves and stop at the first refused one, after moving the ones before it. `rename_dirs` alone renames a directory whose entries are all directories; a directory that also holds files needs `rename_files`, or `rename`, on both paths as well. Configure the same permissions and filters on both paths, or move the contents, and the directory is renamed in one operation.

### What per-directory permissions do not decide

**Where a link leads.** Operations dereference a symbolic link and run under the permissions of the path the client requests, not of the link's target. In the example above SFTPGo refuses a link to `/account/inbound`, to `/account/outbound` or to their contents, since neither those directories nor `/account` grant `create_symlinks`, but a symbolic link already present on the storage is dereferenced. When per-directory permissions wall off sibling directories, keep symbolic-link creation disabled, its default [`symlink_mode`](config-file.md#symbolic-links-and-permissions), and see [How path-based restrictions are evaluated](#how-path-based-restrictions-are-evaluated).

**Whether a name is visible.** Permissions control what a user may *do*. In the example above, `inbound` and `outbound` still appear when listing `/account`, even though every operation there is denied. To hide the names as well, use a [file pattern filter](#file-pattern-filters) with the hide policy, or isolate at the storage layer so the siblings are never exposed.

## File pattern filters

File pattern filters restrict which **names** are listed and transferable within a directory. For a given path you define case-insensitive, shell-like allowed and denied patterns; allowed patterns are evaluated before denied ones. Like permissions, a filter defined for a path also applies to its subdirectories unless a more specific filter is defined.

In the WebAdmin the filters are the entries of the **Per-directory name patterns restrictions** section of the user or group. Each entry has a path, comma-separated allowed patterns, comma-separated denied patterns and a deny policy.

Matching is performed on the file or directory **name**, and applies to both files and directories. Each filter has a deny policy:

| Deny policy | Behavior |
| ----------- | -------- |
| **Default** | A denied file is still shown in directory listings but cannot be uploaded, downloaded, overwritten, or renamed, and it is refused as the source or the destination of a server-side copy. A denied directory keeps being listed and traversed, and its entries are matched one by one; the name itself is refused wherever it is written: created, renamed, removed, or used as a copy destination. |
| **Hide** | The same restrictions apply, the denied item is also removed from directory listings, and reading it reports it as missing rather than refused. Writing to a denied name, creating it, renaming or copying onto it, reports the refusal: a path the client supplies for the first time conceals nothing. May affect performance for very large directories. |

Two rules govern how allowed and denied patterns interact:

- Allowed patterns are checked first; a name that matches one is accepted even if it would also match a denied pattern.
- A non-empty allowed list means **deny by default**: anything that does not match an allowed pattern is denied. An empty allowed list means **allow by default**: everything is accepted except names matching a denied pattern.

A filter describes the names *inside* the path it is defined on, so a filter on `/data` governs the entries of `/data`, not the name `data` itself. Under the **hide** policy the distinction shows up when a directory is read by name: a directory hidden by a filter defined on one of its parents is reported as missing, while a directory whose own filter hides all of its entries is reported as empty.

Listing and transfers follow the same rule, so a filter that hides every name in a directory denies every transfer there as well, uploads included: denying `*` on `/` under the hide policy leaves the user with an empty listing and no way to upload. To keep uploads working, deny the names that must stay hidden and leave the names users upload allowed.

A name filter is matched whenever a directory name comes into existence: when the directory is explicitly created, renamed or copied, and when it is created as a missing parent on behalf of the user. Writing into a directory whose name is denied but that already exists stays allowed, unless the name is hidden. On the cloud storage backends directory entries are derived from object keys and no parent is ever created, so a permitted upload can still bring a denied path prefix into existence there. Use the `create_dirs` permission to control where directories may be created.

A filter is attached to a virtual path and does not follow a directory that is moved: after a rename, the entries the directory carried are governed by whatever applies at the new location. When a filter is defined on the moved tree or on the target tree, or the two paths inherit different filters, the entries are authorized one at a time, as described for [renaming a directory](#how-each-operation-is-checked); with the same filters on both paths the directory is renamed in one operation.

:warning: Matching is on the name, so file pattern filters are a visibility and transfer control, not a content filter. A rule that denies `*.exe` matches the name only, and the same content uploaded as `*.txt` is allowed. Use them to organize what is listed and to block transfers by name; to control which directories a user can reach, rely on storage isolation, virtual folders, or per-directory permissions.

### Examples

**Allow only images in a directory.** Only the matching names can be uploaded, downloaded or listed in `/photos`; `report.txt` is denied because the allowed list is non-empty.

| Field | Value |
| ---------- | ------- |
| Path | `/photos` |
| Allowed patterns | `*.jpg, *.png` |
| Deny policy | Default |

**Block executables everywhere.** Everything is allowed except the denied names. With the default deny policy they remain visible but cannot be transferred; the same content uploaded under an allowed name is accepted, since matching is by name.

| Field | Value |
| ---------- | ------- |
| Path | `/` |
| Denied patterns | `*.exe, *.bat, *.sh` |
| Deny policy | Default |

**Hide sibling directories from listings.** Both names vanish from the listing of `/account`; combine with per-directory permissions to also block access.

| Field | Value |
| ---------- | ------- |
| Path | `/account` |
| Denied patterns | `inbound, outbound` |
| Deny policy | Hide |

**Show only a chosen subtree.** Every top-level name except `public` is hidden, so the user effectively sees only `/public`. Because a filter is inherited by subdirectories, this also hides non-matching names deeper in the tree. A filter defined on a subdirectory takes over for the entries that subdirectory carries; the name of the subdirectory itself keeps matching the patterns of `/`, so list there every directory name that stays visible.

| Field | Value |
| ---------- | ------- |
| Path | `/` |
| Allowed patterns | `public` |
| Deny policy | Hide |

**Inheritance with an override.** A filter on `/` applies to the whole tree; a more specific filter on `/incoming` relaxes or tightens the rule just there. The deepest matching path wins, exactly like permissions.

| Field | Value |
| ---------- | ------- |
| Path | `/` |
| Denied patterns | `*.exe` |
| Deny policy | Hide |

| Field | Value |
| ---------- | ------- |
| Path | `/incoming` |
| Allowed patterns | `*` |
| Deny policy | Default |

## How path-based restrictions are evaluated

Per-directory permissions, [file pattern filters](#file-pattern-filters), allowed and denied [share paths](tutorials/shares.md#restricting-shareable-paths), and every other setting keyed on a path are evaluated against the **virtual path the client requests**, not against the physical location that path ultimately resolves to. The configured restrictions belong to the requested path; they do not follow a redirection in the underlying storage.

When the storage redirects one path to another, through a symbolic link, a Windows junction, a bind mount, a hard link, or any similar operating-system or configuration mechanism, SFTPGo applies the restrictions configured for the path the client named. If `/path1` resolves to `/path2`, the access decision uses what is configured for `/path1`, **even when `/path2` is configured with stricter restrictions**. A tighter configuration on `/path2` does not protect it from access through `/path1`, and the restrictions defined for `/path2` are not consulted. A public share is a boundary of the same kind: it is evaluated on the virtual path requested through the share link, so a symbolic link beneath a shared path is dereferenced and its target is served through the share even when it sits elsewhere in the owner's tree.

The practical consequence is that these restrictions are only as strong as the storage layout beneath them. SFTPGo confines symbolic links to the user's accessible area and disables link creation by default ([`symlink_mode`](config-file.md#symbolic-links-and-permissions)), but what the filesystem itself presents beneath a home directory is served as it appears: a mount point, a Linux bind mount and a special filesystem such as `/proc` are traversed like ordinary directories, see [Local filesystem](localfs.md#symbolic-links). Provision the storage so its directory structure reflects the boundaries you intend, and do not place a link, junction, or mount inside a user's tree that points at a location you restrict elsewhere.

More generally, SFTPGo trusts the storage backend. What the storage reports, the structure it presents, the names it returns, the file types and the metadata it declares, is taken as authoritative, and the restrictions described here are layered on top of it. Content written to the storage outside SFTPGo, by another tool or directly on the filesystem, is served as it is reported.

The [`create_symlinks` permission](#per-directory-permissions) is itself path-based and bidirectional: a directory that grants it can both host links and have its entries made the target of links created elsewhere. The check is on the directory that contains the target, so everything below a directory that grants it can be reached through a link, a subdirectory that restricts operations included, under the permissions of the directory holding the link. A user holding `create_symlinks` in two directories can link one to the other and reach either under the more permissive directory's rights. Do not grant `create_symlinks` on a directory when it, or any directory below it, restricts operations, and see [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) for the full interaction.

## IP and protocol restrictions

You can gate the connection itself, independently of the filesystem:

- **Allowed/denied IP addresses**: restrict the source addresses a user may connect from, in CIDR notation (for example `192.0.2.0/24` or `2001:db8::/32`). An address matching an allowed entry is accepted, an address matching a denied entry alone is refused, and with a non-empty allowed list an address matching neither is refused.
- **Source networks of a public key**: an SSH public key prefixed with the `from` option authenticates from the listed networks, so each key of a user can have its own, see [Restricting a key to source addresses](ssh.md#restricting-a-key-to-source-addresses).
- **Allowed/denied protocols**: limit the user to specific protocols (SFTP, FTP, WebDAV, HTTP).
- **Allowed/denied login methods**: limit which authentication methods the user may use (for example, public key only).
- **Two-factor requirement per protocol**: require an additional one-time code on the selected protocols.

## Time-based access

Access can be limited to specific weekly time windows. Each period defines a day of the week and a `from`/`to` interval in `HH:MM`; the user may log in only during one of the configured periods. A session that is open when the period ends is not guaranteed to be closed at that moment.

## When account changes take effect

An account is loaded from the data provider at login, with the settings of the [groups](groups.md) it belongs to merged in, and the session keeps those settings for as long as it lasts, which can be long after a change; an operation already authorized is not authorized again while it runs. [`token_validation`](config-file.md#http-server) governs whether a Web Client, WebAdmin or REST API token issued before a change stays valid.

Deleting a user through the WebAdmin or the REST API closes the sessions it has open. Updating an account leaves its sessions open unless the update asks otherwise: the WebAdmin user page offers **Disconnect the user after the update**, and the REST API accepts `PUT /users/{username}?disconnect=1`. Changing a group has no equivalent option. Active sessions can be closed at any time from the Connections page or through the REST API. In a cluster, a session served by another instance is closed through a request to that [instance](config-file.md#node).

To apply a change to what is already connected, change the account and then close its sessions.

Saving an account writes the whole record. When two edits reach the same account at the same time, from two administrators or from an administrator and the account's own Web Client, the last one saved wins.

## Choosing the right mechanism

- The user should only ever work inside one subtree => **isolate at the storage layer**: make that subtree the root (home directory for local storage, key prefix for cloud backends). The rest cannot be discovered.
- The user needs the shared parent but with different rights per subdirectory => **per-directory permissions**, optionally combined with the **hide** deny policy to also hide the denied names.
- You want to organize listings or block specific names from being transferred => **file pattern filters**.
- You want to restrict where, how, or when a user connects => **IP, protocol, login method**, and **time-based** restrictions.
