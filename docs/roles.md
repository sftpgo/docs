---
description: "Delegate administration in open-source SFTPGo with roles. Restricted admins can only manage users that share their assigned role."
---

# Roles

Roles can be assigned to users and administrators. Admins with a role are limited administrators, they can only view and manage users with their own role and they cannot have the following permissions:

- manage_admins
- manage_system
- manage_event_rules
- manage_roles
- view_events

Users created by role administrators automatically inherit their role.

Admins without a role are global administrators and can manage all users (with and without a role) and assign a specific role to users.

A role limits which users an administrator manages, not the storage those users reach: an administrator who can add or edit users, or manage folders, sets any home directory, mapped path and storage configuration. Give these permissions only to administrators trusted with all the storage SFTPGo can access.
