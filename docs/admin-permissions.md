---
description: "SFTPGo admin permissions: the grants an administrator can hold, the granular split for groups and folders, and common admin profiles."
---

# Admin Permissions

Each administrator is granted one or more **permissions**. A permission is a string identifier that gates a feature: list users, create groups, scan quotas, view server status, and so on. Permissions can be combined freely to tailor the authority of each admin — from full super-admin access to focused profiles like helpdesk read-only or folder-catalog steward.

A super administrator is configured by granting the wildcard permission `*`, which trumps every other permission.

## Permission reference

| Permission | What it allows |
| ---------- | -------------- |
| `*` | Super-admin. Grants every permission below and access to the features that have no dedicated permission string — administrators and roles, system configuration sections, Event Manager rules and actions, IP allow/deny lists, API keys, retention and metadata checks. |
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

Features without a dedicated permission — admin management, role management, Event Manager rules and actions, system configuration sections (bindings, email, branding, ...), IP allow/deny lists, API keys, retention checks, metadata checks — require the wildcard `*`.

## Granular group and folder permissions

`view_*`, `manage_*`, and `del_*` are independent permissions for both groups and folders. You can grant a read-only role on the group catalog (`view_groups`), a catalog steward role that can add and edit but cannot delete (`view_groups` + `manage_groups`), or a destructive grant alone (`del_groups`).

The catalogs these permissions act on are global. When the administrator carries a role with [resource isolation](roles.md#resource-isolation) enabled, they act on the groups and the folders carrying that role: the administrator reads its own catalog, the groups and folders it creates receive the role, and the storage they may name is bounded by the [storage allowlist](roles.md#storage-allowlist) of the role.

### Cross-permission rule: `view_folders` is required alongside group permissions

Granting `view_groups` or `manage_groups` also requires `view_folders`: a group carries the virtual folders mapped on it, so whoever reads a group reads the names of those folders, and attaching a folder to a group means choosing one from the catalog. The rule keeps the visibility of the two catalogs consistent.

The rule is one-way: granting `view_folders` or `manage_folders` does not require any group permission, since a folder references no group.

A save that violates the rule is refused, from the WebAdmin, the REST API and a backup restore alike, unless the [legacy compatibility flag](#legacy-compatibility-env-var) is set.

### Groups attached automatically on user create

Each administrator can carry a list of groups, configured by a super administrator. When an admin without `view_groups` creates a new user, the configured admin groups are automatically attached as primary, secondary, or membership groups according to each entry's `add_to_users_as` setting. This makes it possible to deploy a tenant-operator profile that provisions users while remaining unable to see or pick groups from the global catalog.

Auto-apply runs on user **create** only: updates preserve the existing groups on the user record. When the admin has `view_groups`, the step is skipped on create too: the admin owns the group selection on the user. For an administrator carrying a role that [isolates its resources](roles.md#resource-isolation), the configured groups must carry the same role.

## Detail pages with read-only access

When an administrator has `view_*` for a resource but lacks the corresponding write permission, the WebAdmin detail page opens in read-only mode:

- The user detail page is read-only for admins that have `view_users` and not `edit_users`.
- The group detail page is read-only for admins that have `view_groups` and not `manage_groups`.
- The folder detail page is read-only for admins that have `view_folders` and not `manage_folders`.

Form inputs and selects are not editable, the submit button is hidden, and accordions remain expandable so the admin can inspect every section.

Read-only pages carry a **Read only** badge in their header.

For remote storage backends (S3, Google Cloud Storage, Azure Blob, SFTP, FTP, HTTP) the read-only page keeps the **Test connection** button: it probes the filesystem configuration exactly as stored, so a view-only admin can verify that credentials and connectivity are still valid — useful for helpdesk profiles diagnosing login failures. Testing a modified configuration requires the corresponding edit permission. For an administrator carrying a role that isolates its resources, a test that sends a filesystem configuration is measured against the role's [storage allowlist](roles.md#storage-allowlist), while a test of the configuration already stored runs whatever the allowlist says.

The user page also shows the [**Effective storage**](groups.md#inspecting-the-effective-storage) panel when the storage that applies differs from the one configured on the user. Every admin who can open the page sees the provider that applies and whether login is denied. The name of the resource providing the storage, its location parameters — the home directory included — and the panel's **Test connection** button require the matching view permission, `view_groups` or `view_folders`: the same permissions that decide whether the page shows the group and folder sections.

The [**Effective configuration**](groups.md#inspecting-the-effective-configuration) page shows the whole user with the group settings applied, so it requires `view_groups` in addition to the permission to open the user page. It is read-only for every admin, edit permissions included, and saves nothing.

## Legacy compatibility env var

`SFTPGO_HOOK__LEGACY_ADMIN_PERMS=1` restores the umbrella semantics where `manage_groups` and `manage_folders` also grant view and delete on the respective catalogs, and admins without `view_groups` / `view_folders` can attach groups and folders to users via the user save endpoints. With this flag enabled the cross-permission validation rule is bypassed too.

The flag is intended as a transition bridge for in-place upgrades from instances that relied on the previous umbrella semantics. The recommended path is to grant the explicit permissions to existing admins and remove the flag. A future major release will retire it.

## Common admin profiles

A few combinations recur often:

- **Super administrator** — `*`. Owns everything, including the features that have no dedicated permission string.
- **Tenant operator** — `view_users` + `add_users` + `edit_users` + `del_users` paired with a [role](roles.md) and an administrator group entry carrying the same role. The operator provisions users that automatically receive the configured tenant group; the group's primary filesystem and folders apply at runtime. With [resource isolation](roles.md#resource-isolation) enabled on the role, the group and folder permissions can be granted as well: the operator curates the catalog of its own role, within the storage its [allowlist](roles.md#storage-allowlist) declares.
- **Helpdesk read-only** — `view_users` + `view_groups` + `view_folders`. Detail pages open in read-only mode; useful for support staff that inspects user records without changing them.
- **Helpdesk with operational actions** — read-only profile above plus `quota_scans` + `close_conns` + `disable_mfa`. Each action stays gated by its own permission and is exposed as a button on the user detail page.
- **Group catalog steward** — `view_groups` + `manage_groups` + `view_folders`, optionally `del_groups`. Curates the group catalog; cannot manage users directly.
- **Folder catalog steward** — `view_folders` + `manage_folders`, optionally `del_folders`. Curates the folder catalog only.
- **Provisioning automation** — `add_users` alone, with the administrator groups configured. The auto-apply step attaches the configured groups; the bot creates users without listing the catalog.

## Where to grant permissions

- **WebAdmin** — open the admin detail page and select the desired permissions in the multi-select.
- **REST API** — set the `permissions` array on the admin payload. See `AdminPermissions` in the [REST API reference](https://sftpgo.com/rest-api){:target="_blank"} for the full schema.
- **Terraform** — set the `permissions` attribute on the `sftpgo_admin` resource. See the [Terraform provider documentation](https://registry.terraform.io/providers/drakkan/sftpgo/latest/docs/resources/admin){:target="_blank"}.

[Role-based administrators](roles.md) carry additional restrictions on top of the permissions you assign — see the Roles page for the full list.
