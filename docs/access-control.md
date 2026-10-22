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

On local storage on Windows a wildcard path ignores the letter case of ASCII letters: the path and the directory are compared with their ASCII letters in lowercase. Write ranges with letters of one case (`[a-z]` or `[A-Z]`): a range that mixes letters with other characters changes meaning once lowercased, so `[A-z]` matches only what `[a-z]` matches and `[Z-a]` matches nothing. Letters outside ASCII are compared as written in the component that holds the wildcard and in the components after it: for a directory whose settings must hold for every spelling of its name, use a literal path, see [Names that reach the same entry](#names-that-reach-the-same-entry). On every other storage a wildcard path is matched as written, letter case included.

When more than one wildcard path matches a directory at the same depth, which one applies is not defined: write wildcard paths that do not overlap. On local storage on Windows the comparison ignores letter case, so two paths that do not overlap elsewhere can overlap there: `/Data/*` and `/da*a/x` both match `/Data/x`.

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

Patterns are stored in lowercase and compared with the name in lowercase, on every platform and storage, following the Unicode case rules: a pattern matches every case variant of a name that Unicode recognizes, including some that a file system can keep as distinct names, such as the Kelvin sign `K` for `k`. Write letters in lowercase, ranges included (`[a-z]`): a range that mixes uppercase letters with other characters changes meaning once lowercased, `[Z-a]` becomes `[z-a]`, which matches nothing.

A filter describes the names *inside* the path it is defined on, so a filter on `/data` governs the entries of `/data`, not the name `data` itself. Under the **hide** policy the distinction shows up when a directory is read by name: a directory hidden by a filter defined on one of its parents is reported as missing, while a directory whose own filter hides all of its entries is reported as empty.

Listing and transfers follow the same rule, so a filter that hides every name in a directory denies every transfer there as well, uploads included: denying `*` on `/` under the hide policy leaves the user with an empty listing and no way to upload. To keep uploads working, deny the names that must stay hidden and leave the names users upload allowed.

A name filter is matched whenever a directory name comes into existence: when the directory is explicitly created, renamed or copied, and when it is created as a missing parent on behalf of the user. Writing into a directory whose name is denied but that already exists stays allowed, unless the name is hidden. On the cloud storage backends directory entries are derived from object keys and no parent is ever created, so a permitted upload can still bring a denied path prefix into existence there. Use the `create_dirs` permission to control where directories may be created.

A filter is attached to a virtual path and does not follow a directory that is moved: after a rename, the entries the directory carried are governed by whatever applies at the new location. When a filter is defined on the moved tree or on the target tree, or the two paths inherit different filters, the entries are authorized one at a time, as described for [renaming a directory](#how-each-operation-is-checked); with the same filters on both paths the directory is renamed in one operation.

:warning: File pattern filters match names, not content: a rule that denies `*.exe` matches the name only, and the same content uploaded as `*.txt` is allowed. Use them to organize what is listed and to block transfers by name, and [ICAP](tutorials/eventmanager-icap.md) or hooks to inspect content.

A pattern matches a name when their lowercase forms match, on every storage. A storage can also resolve other spellings to the same file. On local storage on Windows, short names in their usual form, with a `~`, and names ending with a dot or a space are refused, so a file is reached by its long name; other spellings that the storage treats as equal although their lowercase forms differ, such as another Unicode normalization of the same letters, or a short name that a remote server resolves to the file, are not matched by a pattern written for the long name. To keep files out of a user's reach, place them where the user has no access, through storage isolation, virtual folders or per-directory permissions, rather than relying on a pattern; see [Names that reach the same entry](#names-that-reach-the-same-entry) for how each storage resolves spellings.

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

Per-directory permissions, [file pattern filters](#file-pattern-filters), allowed and denied [share paths](tutorials/shares.md#restricting-shareable-paths), and every other setting keyed on a path are evaluated against the **virtual path the client requests**, not against the physical location that path ultimately resolves to. The configured restrictions belong to the requested path; they do not follow a redirection in the underlying storage. On local storage on Windows, the spellings of a name that the storage resolves to the same entry, such as its letter case variants, count as that name, see [Names that reach the same entry](#names-that-reach-the-same-entry).

When the storage redirects one path to another, through a symbolic link, a Windows junction, a bind mount, a hard link, or any similar operating-system or configuration mechanism, SFTPGo applies the restrictions configured for the path the client named. If `/path1` resolves to `/path2`, the access decision uses what is configured for `/path1`, **even when `/path2` is configured with stricter restrictions**. A tighter configuration on `/path2` does not protect it from access through `/path1`, and the restrictions defined for `/path2` are not consulted. A public share is a boundary of the same kind: it is evaluated on the virtual path requested through the share link, so a symbolic link beneath a shared path is dereferenced and its target is served through the share even when it sits elsewhere in the owner's tree.

The practical consequence is that these restrictions are only as strong as the storage layout beneath them. SFTPGo confines symbolic links to the user's accessible area and disables link creation by default ([`symlink_mode`](config-file.md#symbolic-links-and-permissions)), but what the filesystem itself presents beneath a home directory is served as it appears: a mount point, a Linux bind mount and a special filesystem such as `/proc` are traversed like ordinary directories, see [Local filesystem](localfs.md#symbolic-links). Provision the storage so its directory structure reflects the boundaries you intend, and do not place a link, junction, or mount inside a user's tree that points at a location you restrict elsewhere.

More generally, SFTPGo trusts the storage backend. What the storage reports, the structure it presents, the names it returns, the file types and the metadata it declares, is taken as authoritative, and the restrictions described here are layered on top of it. Content written to the storage outside SFTPGo, by another tool or directly on the filesystem, is served as it is reported.

A requested path uses both `/` and `\` as directory separators, on every protocol, and restrictions apply to the path as read, so the names SFTPGo creates never contain `\`. An entry whose name contains `\` is left out of directory listings, since a request reads `\` as a separator: removing the directory that holds it deletes the rest of its contents and keeps the directory. Rename or delete such an entry directly on the storage.

The [`create_symlinks` permission](#per-directory-permissions) is itself path-based and bidirectional: a directory that grants it can both host links and have its entries made the target of links created elsewhere. The check is on the directory that contains the target, so everything below a directory that grants it can be reached through a link, a subdirectory that restricts operations included, under the permissions of the directory holding the link. A user holding `create_symlinks` in two directories can link one to the other and reach either under the more permissive directory's rights. Do not grant `create_symlinks` on a directory when it, or any directory below it, restricts operations, and see [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) for the full interaction.

### Names that reach the same entry

A storage can resolve several spellings of a name to one file or directory. On Windows, by default, names that differ only in letter case reach the same entry, and so do the short name the volume generates for a long name (`PASTAR~1` for `Pasta Riservata`) and a name followed by dots or spaces. Remote servers, network shares and object storage services follow naming rules of their own.

**Local storage on Windows.** This covers the local filesystem and its [encrypted variant](dare.md). The settings configured for a path apply to every spelling that reaches the same file or directory: per-directory permissions, file pattern filters and allowed and denied share paths configured for `/private` apply to `/PRIVATE` and `/Private/sub` as well, and those configured for `/étude` apply to `/ÉTUDE` when the storage resolves it to that directory. Short names in their usual form, with a `~`, and names ending with a dot or a space are refused for every operation, so files and directories are reached by their long name: a short name is reported as missing, a name ending with a dot or a space as not allowed.

Which names reach the same entry is decided by the storage, from what the volume or share reports, see [File names on Windows](localfs.md#file-names-on-windows). SFTPGo asks the storage only when both the requested name and the configured path contain letters outside ASCII: if the storage treats a name as equal to a configured path where only one of the two contains such letters, that name reaches the directory with its own settings, not with those of the configured path.

**Every other storage**, local storage on Linux included, SFTP, FTP and HTTP storage, S3, Google Cloud Storage and Azure Blob, matches restrictions against the requested path as written, and the storage decides which spellings reach the same entry. Storage that matches names exactly has no other spelling to reach. Storage that does not, such as a Linux file system or directory set to ignore letter case, a network share mounted on Linux, a remote server running on Windows or an object storage service that keeps its keys on a case-insensitive file system, lets another spelling of a restricted directory reach it with the settings of the path as written. To make restrictions independent of the storage's naming rules, assign them at folder level: map each area that needs different settings to its own [virtual folder](virtual-folders.md). A virtual folder is reached only through its exact mount path, so the folder and the settings assigned to its mount path hold whatever spelling a client uses.

On local storage on Windows, keep the configuration unambiguous:

- Use one spelling per directory in permissions, file pattern filters, share paths and mount paths, across user and group settings too. When two configured paths name the same directory with different spellings, the settings of one of them apply, and which one is not defined.
- Configured paths are compared as written: a path written with a short name, through a link or a junction, or as another spelling of a path configured elsewhere is a different path for every comparison. Write each path with long names, in the spelling the storage holds.
- A spelling of a virtual folder's mount path other than the configured one names the directory with that name on the parent's storage, not the folder, as on every platform. When settings are defined for the mount path or for a path below it, such a spelling is refused.
- Renaming a directory to a name that the storage cannot yet compare with the configured paths of the target directory is refused, since the target does not exist before the rename: this is the case for a name with letters outside ASCII next to configured paths with letters outside ASCII. Create the directory and move the contents into it.
- Every directory named by a configured path must be readable by the account SFTPGo runs as. When one of them cannot be read, the other directories with names outside ASCII next to it are refused, together with what they contain.
- A directory set to distinguish letter case (`fsutil file setCaseSensitiveInfo`) is treated like the rest of the volume: a name that differs from a configured path only in the case of ASCII letters gets that path's settings, even where the directory holds it as a separate entry. Give directories that need different settings distinct names, not case variants.
- A short name is recognized by its form: up to eight characters, with an optional extension of up to three, containing a `~` followed by a letter or a digit, the form of the short names Windows and Samba generate with a numeric or hashed tail. A name of that form is refused when it reaches an entry whose long name is different, so a file whose own name has that form stays reachable. A short name in another form, such as one assigned explicitly by an administrator or one a volume generates without `~`, is not recognized: do not assign short names, and keep short-name generation disabled on volumes that generate them without `~`.
- For a configured path with letters outside ASCII, the storage can compare a spelling with it only once the directory exists: an operation authorized while the directory is missing, then created by another session under another spelling, runs with the settings of the parent.

## Copy and move onto the same entry

SFTPGo refuses a copy, a move or an Event Manager action that would overwrite or replace its own source, also when the target is another name of it, such as a spelling the storage resolves to it, a hard link, a symbolic link, or a directory reached through a link or a mount, as far as the storage can tell (see [Names that reach the same entry](#names-that-reach-the-same-entry)):

- local storage, on every platform: from the identity the storage reports for each file and directory;
- SFTP, FTP and HTTP storage: by the path as written, the remote server decides which names are equivalent;
- S3, Google Cloud Storage and Azure Blob: by the object key as written, and two folders are the same storage only when they name the same bucket or container.

Keep the source and the target of copies, moves and actions in separate directories, and do not map two virtual folders onto overlapping locations of the same storage, directly, through a link, or through overlapping buckets, containers or prefixes.

A rename that only changes the spelling of a name, such as `report.txt` to `Report.txt`, renames the entry in place: it needs no `overwrite` permission and leaves the quota unchanged.

Limits:

- Storage that does not report a unique, persistent identity for each file cannot tell two names of one file apart: a copy of a file onto another name of itself is then not refused and empties the file.
- A copy into a directory inside its own source under another name stops with an error when it reaches the target, keeping what was copied until then; where the source cannot be read through the names being written, the copy stops at the maximum directory depth instead.
- A directory moved into a virtual folder mapped inside the directory itself stops with an error; the entries moved until then stay inside the source tree.
- The checks run before the operation: an entry another session creates or replaces in the meantime is not seen. Symbolic-link creation is disabled by default, see [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions).

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
