---
description: "SFTPGo users: credentials, storage backend and root directory, permissions, connection restrictions, quota and bandwidth limits, groups and lifecycle."
---

# Users

A user is an account that logs in over SFTP, FTP, WebDAV or the Web Client and works on the storage configured for it. Every setting that decides what an account can reach and do belongs to the user or to the [groups](groups.md) it is a member of; this page maps those settings and links the pages that describe each of them.

Users are created and edited in the WebAdmin, through the [REST API](rest-api.md) and the Terraform provider, imported from a backup, or provisioned at login by a [pre-login hook](dynamic-user-mod.md), an [external authentication hook](external-auth.md), an authentication plugin or an [OpenID Connect](oidc.md) identity provider. Whatever creates them, the stored account is the same object with the same rules.

A working account needs a username, a way to authenticate, a storage with its root directory and the permissions granted on `/`. Everything else on this page refines it. A user with neither a password nor a public key is valid: it authenticates with an SSH certificate or through OpenID Connect.

## Identity and credentials

- **Username**: unique across the installation and fixed once the user is created. The allowed characters are set by `naming_rules` in the [configuration](config-file.md#data-provider).
- **Status** and **Expiration**: an inactive or expired user is refused on every protocol. The other settings stay stored, so an account is disabled and re-enabled without losing anything. A primary group sets the expiration through **Expires in**, as days from the creation of the user.
- **Password**: stored hashed. **Password expiration** is the number of days a password stays valid; **Require password change** asks the user to set a new password from the Web Client, and the other protocols wait until it is done. Algorithms and validation rules are in [Password management](password.md).
- **Public keys**: SSH public keys in the `authorized_keys` format: RSA from 2048 bits, ECDSA, Ed25519 and FIDO security keys (`sk-*`). A `from="<CIDR>,..."` option restricts a key to the listed networks, see [Restricting a key to source addresses](ssh.md#restricting-a-key-to-source-addresses). SSH certificates are configured on the server, see [SSH authentication](ssh.md#authentication).
- **TLS certificates**: PEM certificates that log the user in on FTP and WebDAV bindings requesting a client certificate. **TLS username** set to `CN` accepts instead any trusted certificate whose Common Name equals the username. See [FTP](ftp.md#authentication) and [WebDAV](webdav.md#authentication).
- **Two-factor authentication**: a time-based one-time code, enrolled by the user from the Web Client. **Require 2FA for** makes it mandatory on the selected protocols. See the [two-factor authentication](tutorials/two-factor-authentication.md) tutorial.
- **Email** and **Additional emails**: the addresses for password resets, one-time codes and notifications.
- **Role**: assigns the user to a [role](roles.md), so that the administrators of that role manage it.
- **Description**: free text, shown to the user in the Web Client profile. **Additional info**: free text for the administrator alone.

## Storage

Every user works in a file system of its own, seen as `/` from every protocol. It is built from two kinds of pieces:

- The **root filesystem** is the storage selected in the **Storage** section of the user: a directory on the local disk, plain or encrypted, a bucket or container with an optional key prefix on S3, Google Cloud Storage or Azure Blob, or a path on a remote SFTP, FTP or HTTP server. The user sees it as `/` and reaches nothing above it. **None** takes the storage from the primary group, see [Users with storage "None"](groups.md#users-with-storage-none). Each backend has its own page: [local filesystem](localfs.md), [encrypted](dare.md), [S3](s3.md), [Google Cloud Storage](google-cloud-storage.md), [Azure Blob](azure-blob-storage.md), [SFTP](sftpfs.md), [FTP](ftpfs.md), [HTTP](httpfs.md).
- **Virtual folders** are additional locations mounted at a chosen path of that file system. A folder is defined once, with a backend of its own that can differ from the root's, and mapped to any number of users or groups: mapped to several users it becomes a shared area. Each mapping has a quota of its own or counts within the user's. See [Virtual folders](virtual-folders.md).

For example, a user can be given a private local directory as root and several locations mounted into it:

| Path | Storage | Notes |
| ---- | ------- | ----- |
| `/` | Local disk, `/srv/sftpgo/alice` | Root filesystem, private to the user. |
| `/shared` | S3 bucket `company-data`, key prefix `teams/sales/` | Virtual folder mapped to every member of the team. |
| `/reports` | S3 bucket `company-data`, key prefix `reports/2026/` | Virtual folder on the same bucket, another prefix. |
| `/backups` | Azure Blob container `backups` | Virtual folder with a quota of its own. |
| `/archive` | Directory on a remote SFTP server | Virtual folder. |

The user browses one tree, and the permissions, name patterns and quota described below apply to its paths whatever backend serves them.

The **Root directory** field (`home_dir` in the REST API, "home directory" in these pages) is a directory on the host running SFTPGo. With the local disk storages it is the directory served as `/`, created at the first login when missing. With any other storage it is a working directory for SFTPGo: the help text of the field says when it can be left blank, and the user page shows the value SFTPGo derived from the username.

:information_source: What the user can reach within the storage is decided by the settings below and by the storage layout: see [Access control](access-control.md) for how the layers compose, and [symbolic links](localfs.md#symbolic-links) for root directories that already contain links.

## What the user may do

- **Permissions**: the operations allowed in `/`, refined by **Per-directory permissions** for the paths that need different rights. See [Per-directory permissions](access-control.md#per-directory-permissions).
- **Per-directory name patterns restrictions**: which file and directory names are listed and transferable, hidden or refused according to the deny policy. See [File pattern filters](access-control.md#file-pattern-filters).
- **Initial directory**: where a session starts, within the root; it restricts nothing. SFTP and FTP sessions start there, the Web Client opens on it and the user REST API resolves relative paths from it; WebDAV starts at the root.
- **Anonymous user**: the account lists and downloads only, over FTP and WebDAV, with the password as the only login method, whatever else is configured.

## How the user may connect

- **Denied protocols**, **Allowed IP/Mask**, **Denied IP/Mask** and **Access time restrictions** gate the connection. See [IP and protocol restrictions](access-control.md#ip-and-protocol-restrictions) and [Time-based access](access-control.md#time-based-access).
- **Denied login methods**: the credentials this user may not use among the ones the binding accepts: password, public key, keyboard-interactive, TLS certificate and, for SSH, the two-step combinations such as public key followed by password. Denying the single SSH methods and leaving a combination requires two-step authentication, see [Multi-step authentication](ssh.md#multi-step-authentication).
- **Enforce secure algorithms** and **FTP security**: the first restricts the SFTP sessions of this user to secure host key, key exchange, MAC and cipher algorithms on a server that enables weaker ones too; the second requires TLS for this user on FTP bindings that allow cleartext sessions too. See [FTP security filter](ftp.md#ftp-security-filter).

## Web Client and REST API

From the Web Client a user manages files and shares, the profile (description, email and additional emails, public keys, TLS certificates), the password and two-factor authentication.

- **Web client/REST API** options remove features from this user: writes, document editing, the recursive size of directories, the REST API, each of the self-service changes above, the API key authentication setting, and the sharing options described in [Shares](tutorials/shares.md). An account provisioned by an identity provider or a hook is kept read-only for the user this way.
- **API key authentication**: lets an API key created for this user act as the user in the REST API, see [REST API](rest-api.md).

## Hooks and login checks

- **Hooks**: excludes this user from the check password, pre-login and external authentication hooks. **External auth cache time** and **Pre-login hook cache time** skip the hook for the given seconds after a successful login.
- **Local accounts beside external ones**: a user with the external authentication hook disabled is verified against the credentials stored in SFTPGo, so local accounts coexist with the ones from an [external system](external-auth.md) or an authentication plugin such as LDAP. The OpenID Connect counterpart is described in [OIDC and local credentials](oidc.md#oidc-and-local-credentials).
- **Disable filesystem checks**: skips the existence checks and the automatic creation of the root directory and of the virtual folder directories at login.

## Quota and limits

The **Disk quota and bandwidth limits** section holds:

- **Quota size** and **Quota files**: the storage the user may use, in bytes (the `MB`/`GB`/`TB` suffix is accepted) and number of files. The usage is a counter SFTPGo updates as files are uploaded and deleted, according to `track_quota` in the [configuration](config-file.md#data-provider); a **quota scan**, from the users list or through the REST API, recomputes it from the storage. Virtual folders have a quota of their own or are counted within the user's, see [Virtual folders](virtual-folders.md#quota).
- **Bandwidth UL/DL**: the upload and download speed of the user, in KB/s, shared by every transfer the user has in progress on the same instance. **Per-source bandwidth speed limits** replace them for connections coming from the listed networks.
- **Upload**, **Download** and **Total data transfer**: the amount of data the user may transfer, in MB; the total limit replaces the two individual ones. The usage grows with every transfer and is reset through the REST API, `PUT /api/v2/quotas/users/{username}/transfer-usage`; per-source limits apply to the listed networks.
- **Max sessions**: concurrent sessions and file transfers.
- **Max upload size**: the size of a single upload, in bytes (the `MB`/`GB`/`TB` suffix is accepted).

`0` means no limit everywhere.

:information_source: Quota and data transfer usage are counters kept by SFTPGo: they follow what goes through it and drift when the storage changes in another way or a transfer is interrupted by a restart. Run a quota scan after changing the storage outside SFTPGo or restoring a backup, and schedule one with the [Event Manager](eventmanager.md) where the counters must stay accurate. `delayed_quota_update` in the [configuration](config-file.md#data-provider) batches the updates.

## Groups

A user belongs to at most one primary group and to any number of secondary and membership groups. The primary and the secondary groups contribute settings to the user at login, so the stored user and the configuration that applies can differ; the **Effective configuration** entry of the users list shows the result. **Custom placeholders** are values a group can reference to derive a path per user. See [Groups](groups.md).

## Lifecycle

- Settings are read at login: a session already open keeps the ones it started with, and the user page offers **Disconnect the user after the update** to close its sessions. See [When account changes take effect](access-control.md#when-account-changes-take-effect).
- Deleting a user closes its sessions and removes the account. The files in its root directory and on its storage stay where they are.
- Users are exported and imported with the rest of the configuration through a [backup](data-provider.md#loading-a-dump-from-the-command-line), and created in bulk from a template in the WebAdmin.
