---
description: "Manage SFTPGo users efficiently with groups. Define permissions, virtual folders, bandwidth limits, and policies once, then apply to multiple users."
---

# Groups

Using groups simplifies the administration of multiple accounts by letting you assign settings once to a group, instead of multiple times to each individual user.

A group can carry a [role](roles.md#assigning-the-role-to-groups-and-folders): the **Role** field of the group page is shown to global administrators, and the groups a role administrator creates take its role.

SFTPGo supports the following types of groups:

| Type | Per user | Description |
| ------ | ---------- | ------------- |
| **Primary** | At most one | Provides the base configuration: the storage, the settings the user leaves unset, and additive settings such as virtual folders and permissions, including the ones for `/`. |
| **Secondary** | Any number | Adds virtual folders, permissions, filters and the other additive settings on top of the primary group and of the user's own settings. It cannot provide the `/` path. |
| **Membership** | Any number | Declares membership only: no setting is inherited. It is used where a rule or a policy is expressed in terms of group membership, such as the conditions of [Event Manager](eventmanager.md) rules and the group access of [shares](tutorials/shares.md). |

:warning: SFTPGo groups are completely unrelated to system groups. There is no need to create Linux/Windows groups to use SFTPGo groups.

The settings are merged when the user logs in, primary group first, then the secondary groups. The stored user is not modified: the [effective configuration](#inspecting-the-effective-configuration) shows the result.

## Settings from the primary group

The primary group provides two kinds of settings: the storage, which replaces the user's own, and the settings the user leaves unset.

### Storage

| Storage of the primary group | Effect on a member |
| ---------------------------- | ------------------ |
| Local disk, or None | The member keeps its own storage. A home directory set on the group replaces the member's one. |
| Encrypted local disk, S3, Google Cloud Storage, Azure Blob, SFTP, FTP, HTTP | Replaces the member's storage. A home directory set on the group replaces the member's one. |

[Placeholders](#placeholders) are supported in the home directory, in the key prefix, in the SFTP prefix, in the FTP/HTTP remote directory and in the SFTP/FTP/HTTP username of the group.

#### Users with storage "None"

A user with storage set to "None" declares that its storage comes from the primary group, and saving such a user requires a primary group. The group provides it when it defines one:

- a cloud or remote storage backend on the group applies as it is;
- Local disk and Encrypted local disk apply when the group sets a home directory, which becomes the root of the member;
- with any other group configuration the user has no storage and its logins are denied.

The user's own home directory serves as the local staging area for transfers, like for cloud backends, and gets an appropriate default when left blank.

### Settings used when the user's value is not set

| Setting | Used when | Notes |
| --------- | ----------- | ------- |
| **Max sessions** | The user's value is `0`. | |
| **Quota size / files** | The user's value is `0`. | |
| **Upload / download bandwidth** | The user's value is `0`. | |
| **Upload / download / total data transfer** | The user sets none of the three limits. | |
| **Max upload file size** | The user's value is `0`. | |
| **External auth cache time** | The user's value is `0`. | |
| **FTP security** | The user's value is `0`. | |
| **Default / max shares expiration** | The user's value is `0`. | |
| **Password expiration** | The user's value is `0`. | |
| **Password strength** | The user's value is `0`. | Enforced whenever a password is set: administrator create and update, self-service change, password reset, share password. The system default applies when the resulting value is `0`. Secondary groups are ignored for password rules. See [Password validation](password.md#password-validation). |
| **Password validation rules** | Each field (length, uppers, lowers, digits, specials) is taken from the group when the user's field is `0`. | The system rules apply as a whole set when the resulting policy has no rule at all. |
| **Expires in** | The user has no expiration date. | Sets the expiration as a number of days from the creation of the user. |
| **Starting directory** | The user has none. | [Placeholders](#placeholders) are supported. |
| **TLS username** | Not set for the user. | |
| **Enforce secure algorithms** | Not set for the user. | |
| **Hook overrides** | Not set for the user. | Check password hook disabled, pre-login hook disabled, external auth hook disabled. |
| **Filesystem checks disabled** | Not set for the user. | |
| **Allow API key authentication** | Not set for the user. | |
| **Anonymous user** | Not set for the user. | |

## Settings from primary and secondary groups

The following settings are **additive**: they are appended to the user's configuration from both the primary and the secondary groups, the primary group first.

| Setting | Merge rule |
| --------- | ----------- |
| **Virtual folders** | Added when the user has no folder at the same mount path. The `/` path comes from the primary group alone. [Placeholders](#placeholders) are replaced in the mount path, in the key prefix, in the FTP/HTTP remote directory, in the SFTP/FTP/HTTP username and in the [sub-path and exposed sub-paths](virtual-folders.md#sub-path-mounts). When two secondary groups mount different folders at the same path, the one that applies is not defined and can change between logins: give each path to one group. |
| **Permissions** | Added when the user has no entry for the same path. The `/` path comes from the primary group alone. |
| **File pattern filters** | Added when the user has no filter for the same path. The `/` path comes from the primary group alone. |
| **Per-source bandwidth limits** | Appended to the user's list. |
| **Per-source data transfer limits** | Appended to the user's list. |
| **Allowed / denied IPs** | Appended to the user's list. |
| **Denied login methods** | Appended to the user's list. |
| **Denied protocols** | Appended to the user's list. |
| **Two-factor auth protocols** | Appended to the user's list. |
| **Web client / REST API permissions** | Appended to the user's list. |
| **Share policies** | Appended to the user's list. |
| **Access time restrictions** | Appended to the user's list. |

No settings are inherited from **membership** groups.

## Placeholders

A group setting can be written as a template and resolved per member, so one group serves accounts that need a location of their own. The values come from the user:

| Placeholder | Value |
| ----------- | ----- |
| `%username%` | The username. |
| `%role%` | The [role](roles.md) of the user. |
| `%custom1%` ... `%custom10%` | The **Custom placeholders** listed on the user page, `custom_placeholders` in the REST API, up to ten. The number is the position in the list. |

They apply to the group settings flagged in the tables on this page, to the [virtual folder](virtual-folders.md) mappings a group carries, and to the paths of per-directory permissions and file pattern filters.

`%username%` and `%role%` are values every account carries, so a group templated on them resolves for each of its members. A custom placeholder is a value an account carries only if it was given one, identified by its position in the list: inserting or removing an entry changes what every later placeholder of that account resolves to, and a member without the value cannot log in until it is set. Use `%username%` and `%role%` where the location can be derived from either, and a custom placeholder for a value that cannot, such as a tenant identifier.

When a group setting uses a placeholder the user has no value for, `%role%` for a user without a role or `%customN%` beyond the custom placeholders defined on the user, the login of that user is refused with an error naming the setting: the setting applies once the value is defined. These failures do not count as authentication failures for the defender.

Virtual folder sub-paths are the exception: the mapping whose sub-path the user cannot render is left unmounted for that user, the other mappings apply and the login succeeds. The value is never rendered with the placeholder replaced by an empty string, so `/tenants/%role%` does not fall back to `/tenants`, the directory that holds every tenant. The server log names the mapping left out and the reason.

A group is saved with its settings as written: which placeholders an account resolves depends on the account. To see what a member gets, open its [effective configuration](#inspecting-the-effective-configuration): the page reports the setting the account cannot resolve, the same answer the login gives, and leaves out the mappings left unmounted, so a group templated on a placeholder can be verified against a real member before the first login.

## Inspecting the effective storage

The storage configured on a user is not always the one that applies at login: the primary group can provide it. When the two differ, the user page shows an **Effective storage** panel above the storage selector, reporting the provider that applies, where it comes from, the location parameters (bucket, container, key prefix, endpoint, remote directory), the home directory when the primary group provides it, and "Login is denied" when no storage applies.

For remote backends the panel has its own **Test connection** button, which probes the effective configuration of that user. The button appears when the resolved backend is available on this installation, so a panel reporting "Login is denied" shows the outcome without it. The button in the storage section below probes the configuration entered in the form.

The panel describes the **saved** user. Changing the storage provider or the primary group in the form marks it "Updated on save" and hides its test button; saving refreshes it.

## Inspecting the effective configuration

The panel answers the storage question. For everything else a group provides, quota, bandwidth, permissions, filters, virtual folders, the **Effective configuration** entry in the Actions menu of the users list opens the user with the group settings applied, read-only: the values that apply at login, with the group placeholders resolved. The entry appears for users that belong to a primary or a secondary group, and requires the permission to view groups.

The page is computed when it is opened, so it reflects the groups as they are at that moment. The expiration date a primary group provides through "Expires in" is marked as inherited: it is derived from the user's creation date and is stored nowhere.

Storage keeps its **Test connection** button on this page, probing the effective configuration. When a virtual folder mounted on the root path provides the storage, the Effective storage panel reports that folder and carries the test button for it.

The same view is available through the REST API: `GET /api/v2/users/{username}/effective` returns the user with the group settings applied, in the same format as `GET /api/v2/users/{username}`. It requires the permissions to view users and groups. Like the page, a virtual folder mounted on the root path provides the storage at login and is reported among the virtual folders, while `filesystem` describes the merge of the user and group configurations.

:warning: The response describes a login, not a stored object: sending it back to `PUT /api/v2/users/{username}` would save the inherited values as the user's own.

## Example

You can define the following groups:

- "group1" has a virtual folder mounted on `/vdir1`
- "group2" has a virtual folder mounted on `/vdir2`
- "group3" has a virtual folder mounted on `/vdir3`

If you create users with a virtual folder mounted on `/vdir` and make them members of all three groups, they have virtual folders at `/vdir`, `/vdir1`, `/vdir2`, and `/vdir3`. If a user already has a virtual folder at `/vdir1`, the group's folder for that path is ignored.

See the [Groups tutorial](tutorials/groups-example.md) for a step-by-step example with screenshots.
