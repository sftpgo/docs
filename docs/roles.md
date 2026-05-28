---
description: "Delegate SFTPGo administration with roles. Restricted admins can only manage users that share their assigned role."
---

# Roles

Roles enable delegated administration: you can create restricted administrators who can only manage users that share their assigned role.

Roles control _which users_ an administrator can see and manage; [Admin Permissions](admin-permissions.md) control _which features_ that administrator can use. The two dimensions compose freely.

## How roles work

A **role** is a named label that can be assigned to both users and administrators. Assigning a role to an administrator makes them a **role-based administrator** — they can only view and manage users with the same role.

When a role-based administrator creates a new user, the user **automatically inherits** that administrator's role. This ensures that delegated administrators can only manage the users they create.

## Role-based vs global administrators

| | Global administrator | Role-based administrator |
| -- | --------------------- | -------------------------- |
| **Role assigned** | None | One role |
| **User visibility** | All users (with or without roles) | Only users with the same role |
| **Can create users** | Yes (any role or no role) | Yes (users always inherit the admin's role) |
| **Can hold the wildcard `*` permission** | Yes | No |
| **Event search results** | All events | Filtered to the admin's role |

## Permission model

A role-based administrator cannot be granted the wildcard `*` permission. The save is rejected with a validation error if you try.

Every other permission in the [Admin Permissions](admin-permissions.md) reference can be assigned to a role-based administrator. Operations that act on users — `add_users`, `edit_users`, `del_users`, `view_users`, `quota_scans`, `disable_mfa` on users — are automatically scoped to users that share the administrator's role.

Features that require `*` — admin management, role management, Event Manager rules and actions, system configuration sections, IP allow/deny lists, API keys, retention checks, dump/restore — are therefore unreachable for role-based administrators.

## Scoping behavior per permission

| Permission | Scoping for role-based administrator |
| ---------- | ------------------------------------ |
| `add_users`, `edit_users`, `del_users`, `view_users` | Operates only on users that share the admin's role. The role on the user record is forced to match the admin's role on create and on update. |
| `quota_scans` | User quota scans target only users sharing the admin's role. |
| `disable_mfa` | Can be applied only to users sharing the admin's role. |
| `view_conns`, `close_conns` | Lists and closes only connections from users sharing the admin's role. |
| `view_events` | Filesystem, provider, and log event searches are filtered to the admin's role. |
| `view_groups`, `manage_groups`, `del_groups`, `view_folders`, `manage_folders`, `del_folders` | Group and folder catalogs are currently global (shared across roles). A role-based administrator with these permissions sees and can manage every group and folder. |
| `view_status` | Server status is global; no scoping applies. |
| `view_defender`, `manage_defender` | The defender blocklist is global (per IP, not per user); a role-based administrator sees and can clear every entry. |

## Typical use case

Roles are useful when you have multiple teams or departments that need independent user management:

1. Create a role for each team (e.g., `finance`, `engineering`).
2. Create an administrator for each team lead and assign the corresponding role.
3. Each team lead can create and manage their own users, but cannot see or affect users belonging to other teams.

Global administrators retain full visibility and control across all roles.
