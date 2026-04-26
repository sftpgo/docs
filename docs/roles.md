---
description: "Delegate SFTPGo administration with roles. Restricted admins can only manage users that share their assigned role."
---

# Roles

Roles enable delegated administration: you can create restricted administrators who can only manage users that share their assigned role.

## How roles work

A **role** is a named label that can be assigned to both users and administrators. Assigning a role to an administrator makes them a **role-based administrator** — they can only view and manage users with the same role.

When a role-based administrator creates a new user, the user **automatically inherits** that administrator's role. This ensures that delegated administrators can only manage the users they create.

## Role-based vs global administrators

| Global administrator | Role-based administrator |
| -- | --------------------- | -------------------------- |
| **Role assigned** | None | One role |
| **User visibility** | All users (with or without roles) | Only users with the same role |
| **Can create users** | Yes (any role or no role) | Yes (users inherit the admin's role) |
| **Can manage admins** | Yes | No |
| **Can manage event rules** | Yes | No |
| **Can manage roles** | Yes | No |
| **Can manage system settings** | Yes | No |
| **Can view events** | Yes | No |

## Restricted permissions

Role-based administrators cannot have the following permissions:

- `manage_admins`
- `manage_system`
- `manage_event_rules`
- `manage_roles`
- `view_events`

If any of these permissions are assigned to an administrator with a role, they are silently ignored.

## Typical use case

Roles are useful when you have multiple teams or departments that need independent user management:

1. Create a role for each team (e.g., `finance`, `engineering`).
2. Create an administrator for each team lead and assign the corresponding role.
3. Each team lead can create and manage their own users, but cannot see or affect users belonging to other teams.

Global administrators retain full visibility and control across all roles.
