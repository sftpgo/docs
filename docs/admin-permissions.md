---
description: "SFTPGo admin permissions: the full list of grants you can assign to administrators, the granular split for groups and folders, the compatibility flag, and common admin profiles."
---

# Admin Permissions

Each administrator is granted one or more **permissions**. A permission is a string identifier that gates a feature: list users, create groups, scan quotas, view server status, and so on. Permissions can be combined freely to tailor the authority of each admin — from full super-admin access to focused roles like helpdesk read-only or folder-catalog steward.

A super administrator is configured by granting the wildcard permission `*`, which trumps every other permission.

## Permission reference

| Permission | What it allows |
| ---------- | -------------- |
| `*` | Super-admin. Grants every permission below and unlocks the features that have no dedicated permission string — administrators and roles, system configuration sections, Event Manager rules and actions, IP allow/deny lists, API keys, retention and metadata checks. |
| `add_users` | Create new users. |
| `edit_users` | Update existing users. |
| `del_users` | Delete users. |
| `view_users` | List users and open the user detail page. |
| `view_groups` | List groups, open the group detail page, and assign groups to users when saving a user. |
| `manage_groups` | Add and edit groups. |
| `del_groups` | Delete groups. |
| `view_folders` | List virtual folders, open the folder detail page, and assign folders to users when saving a user. |
| `manage_folders` | Add and edit virtual folders. Also required to update folder quota usage. |
| `del_folders` | Delete virtual folders. |
| `view_conns` | List active connections. |
| `close_conns` | Close active connections. |
| `view_status` | View the server status page. |
| `quota_scans` | Start quota scans for users and folders, and view active scans. |
| `view_defender` | List the dynamic blocklist. |
| `manage_defender` | Remove entries from the dynamic blocklist. |
| `view_events` | View and search filesystem and provider events. |
| `disable_mfa` | Disable two-factor authentication for users. Resetting two-factor authentication on another administrator account requires the wildcard `*`. |

Features without a dedicated permission — admin management, role management, Event Manager rules and actions, system configuration sections (bindings, email, branding, …), IP allow/deny lists, API keys, retention checks, metadata checks — require the wildcard `*`.

## Granular group and folder permissions

`view_*`, `manage_*`, and `del_*` are independent permissions for both groups and folders. You can grant a read-only role on the group catalog (`view_groups`), a catalog steward role that can add and edit but cannot delete (`view_groups` + `manage_groups`), or a destructive grant alone (`del_groups`).

### Cross-permission rule: `view_folders` is required alongside group permissions

Granting `view_groups` or `manage_groups` also requires `view_folders`. The reason is that a `Group` response embeds its `virtual_folders` mappings, and the WebAdmin add/edit group page renders a dropdown of folder names so the admin can attach them to the group. Anyone who can see a group can therefore see the folder names it references; the validation rule keeps the catalog visibility consistent.

The rule is one-way: granting `view_folders` or `manage_folders` does not require any group permission. Folders do not reference groups in any response.

A save attempt that violates the rule is rejected with a validation error.

### Auto-apply of Admin.Groups on user create

Each administrator can have an associated list of groups (`Admin.Groups`) configured by a super administrator. When an admin without `view_groups` creates a new user, the configured admin groups are automatically attached as primary, secondary, or membership groups according to each entry's `add_to_users_as` setting. This makes it possible to deploy a tenant-operator profile that provisions users while remaining unable to see or pick groups from the global catalog.

Auto-apply runs on user **create** only — updates preserve the existing groups on the user record. When the admin has `view_groups`, the step is skipped on create too: the admin owns the group selection on the user.

## Detail pages with read-only access

When an administrator has `view_*` for a resource but lacks the corresponding write permission, the WebAdmin detail page opens in read-only mode:

- The user detail page is read-only for admins that have `view_users` and not `edit_users`.
- The group detail page is read-only for admins that have `view_groups` and not `manage_groups`.
- The folder detail page is read-only for admins that have `view_folders` and not `manage_folders`.

Form inputs and selects are not editable, the submit button is hidden, and accordions remain expandable so the admin can inspect every section.

## Legacy compatibility env var

`SFTPGO_HOOK__LEGACY_ADMIN_PERMS=1` restores the umbrella semantics where `manage_groups` and `manage_folders` also grant view and delete on the respective catalogs, and admins without `view_groups` / `view_folders` can attach groups and folders to users via the user save endpoints. With this flag enabled the cross-permission validation rule is bypassed too.

The flag is intended as a transition bridge for in-place upgrades from instances that relied on the previous umbrella semantics. The recommended path is to grant the explicit permissions to existing admins and remove the flag. A future major release will retire it.

## Common admin profiles

A few combinations recur often:

- **Super administrator** — `*`. Owns everything, including the features that have no dedicated permission string.
- **Tenant operator** — `view_users` + `add_users` + `edit_users` + `del_users` paired with a [role](roles.md) and an `Admin.Groups` entry. The operator provisions users that automatically receive the configured tenant group; the group's primary filesystem and folders apply at runtime.
- **Helpdesk read-only** — `view_users` + `view_groups` + `view_folders`. Detail pages open in read-only mode; useful for support staff that inspects user records without changing them.
- **Helpdesk with operational actions** — read-only profile above plus `quota_scans` + `close_conns` + `disable_mfa`. Each action stays gated by its own permission and is exposed as a button on the user detail page.
- **Group catalog steward** — `view_groups` + `manage_groups` + `view_folders`, optionally `del_groups`. Curates the group catalog; cannot manage users directly.
- **Folder catalog steward** — `view_folders` + `manage_folders`, optionally `del_folders`. Curates the folder catalog only.
- **Provisioning automation** — `add_users` alone, with `Admin.Groups` configured. The auto-apply step attaches the configured groups; the bot creates users without listing the catalog.

## Where to grant permissions

- **WebAdmin** — open the admin detail page and select the desired permissions in the multi-select.
- **REST API** — set the `permissions` array on the admin payload. See `AdminPermissions` in the [REST API reference](https://sftpgo.com/rest-api){:target="_blank"} for the full schema.
- **Terraform** — set the `permissions` attribute on the `sftpgo_admin` resource. See the [Terraform provider documentation](https://registry.terraform.io/providers/drakkan/sftpgo/latest/docs/resources/admin){:target="_blank"}.

[Role-based administrators](roles.md) carry additional restrictions on top of the permissions you assign — see the Roles page for the full list.
