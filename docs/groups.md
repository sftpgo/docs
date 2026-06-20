---
description: "Manage SFTPGo users efficiently with groups. Define permissions, virtual folders, bandwidth limits, and policies once, then apply to multiple users."
---

# Groups

Using groups simplifies the administration of multiple accounts by letting you assign settings once to a group, instead of multiple times to each individual user.

SFTPGo supports the following types of groups:

| Type | Per user | Description |
| ------ | ---------- | ------------- |
| **Primary** | At most one | Provides the base configuration. Primary group settings are merged first and can override user defaults. |
| **Secondary** | Any number | Adds incremental settings (virtual folders, permissions, filters) on top of the primary group and user settings. Cannot override the root path (`/`). |
| **Membership** | Any number | Indicates group membership only. No settings are inherited. |

:warning: SFTPGo groups are completely unrelated to system groups. There is no need to create Linux/Windows groups to use SFTPGo groups.

## Settings inherited from the primary group

The following settings are taken from the primary group when the corresponding user value is not set (zero, empty, or unset):

| Setting | Merge rule | Notes |
| --------- | ----------- | ------- |
| **Home directory** | Replaces the user's home dir if set in the group. | `%username%` and `%role%` placeholders are supported. |
| **Filesystem config** | Replaces the user's filesystem if the group's provider is different from "local". | `%username%` and `%role%` placeholders are replaced in the key prefix, the FTP/HTTP remote directory, and the SFTP/FTP/HTTP username. |
| **Max sessions** | Used if the user's value is `0`. |
| **Quota size / files** | Used if the user's value is `0`. |
| **Upload / download bandwidth** | Used if the user's value is `0`. |
| **Upload / download / total data transfer** | Used if the user has no main data transfer limits set. |
| **Max upload file size** | Used if the user's value is `0`. |
| **External auth cache time** | Used if the user's value is `0`. |
| **FTP security** | Used if the user's value is `0`. |
| **Default / max shares expiration** | Used if the user's value is `0`. |
| **Password expiration** | Used if the user's value is `0`. |
| **Password strength** | Used if the user's value is `0`. | Enforced whenever a password is set (admin create/update, user self-change, password reset, share password). Falls back to the system-level default if both the user and the primary group set it to `0`. Secondary groups are ignored for password rules. See [Password validation](password.md#password-validation). |
| **Password validation rules** | Each field (length, uppers, lowers, digits, specials) is used if the user's corresponding field is `0`. | Falls back to the system-level default for any field that is `0` at both the user and primary-group level. |
| **Expires in** | Sets the user's expiration date (days from creation) if the user has no expiration set. |
| **Starting directory** | Used if the user has no starting directory set. | `%username%` and `%role%` placeholders are supported. |
| **TLS username** | Used if not set for the user. |
| **Enforce secure algorithms** | Used if not set for the user. |
| **Hook overrides** | Used if not set for the user. | Check password hook disabled, pre-login hook disabled, external auth hook disabled. |
| **Filesystem checks disabled** | Used if not set for the user. |
| **Allow API key authentication** | Used if not set for the user. |
| **Anonymous user** | Used if not set for the user. |

## Settings inherited from primary and secondary groups

The following settings are **additive** — they are appended to the user's configuration from both the primary and secondary groups. The primary group is always merged first.

| Setting | Merge rule |
| --------- | ----------- |
| **Virtual folders** | Added if the user does not already have a folder at the same virtual path. The `/` path is only inherited from the primary group. `%username%` and `%role%` placeholders are replaced in the virtual path, key prefix, FTP/HTTP remote directory, and SFTP/FTP/HTTP username. |
| **Permissions** | Added if the user does not already have permissions for the same path. The `/` path is only inherited from the primary group. |
| **File patterns** | Added if the user does not already have patterns for the same path. The `/` path is only inherited from the primary group. |
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

## Example

You can define the following groups:

- "group1" has a virtual directory mounted on `/vdir1`
- "group2" has a virtual directory mounted on `/vdir2`
- "group3" has a virtual directory mounted on `/vdir3`

If you create users with a virtual directory mounted on `/vdir` and make them members of all three groups, they will have virtual directories at `/vdir`, `/vdir1`, `/vdir2`, and `/vdir3`. If a user already has a virtual directory at `/vdir1`, the group's folder for that path is ignored.

:warning: If the same virtual path is defined in more than one secondary group, the behavior is undefined — the folder mounted at that path may change between logins.

See the [Groups tutorial](tutorials/groups-example.md) for a step-by-step example with screenshots.
