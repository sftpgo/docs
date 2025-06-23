# Roles

Roles can be assigned to users and administrators. Administrators with an assigned role are considered limited administrators: they can view and manage only the users who share their role and they cannot have the following permissions:

- manage_admins
- manage_system
- manage_event_rules
- manage_roles
- view_events

When a user is created by a role-based administrator, the user automatically inherits that administrator’s role.

Administrators without a role are global administrators: they have full access to manage all users (with or without a role) and can assign roles to other users.
