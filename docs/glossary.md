---
description: "Glossary of SFTPGo-specific terms: groups, virtual folders, event rules, placeholders, roles, shares, plugins, and more."
---

# Glossary

A reference of SFTPGo-specific terms. Use it to quickly disambiguate concepts that look similar but behave differently.

## Accounts

**Admin**
: A privileged account that manages the server (users, groups, folders, rules, configuration). Admins sign in to the [WebAdmin](web-interfaces.md#webadmin) or drive the [REST API](rest-api.md). Admins are stored separately from users and cannot transfer files.

**User**
: An end-user account that signs in through one of the supported transfer protocols (SFTP, FTP/S, WebDAV, HTTPS, WebClient). Each user has a home directory, permissions, quotas, and an optional filesystem configuration.

**Role**
: A label attached to users and admins that scopes who can manage whom and restricts resource visibility. See [Roles](roles.md).

## Groups

**Group**
: A reusable set of settings that can be applied to many users. Groups are unrelated to operating-system groups. See [Groups](groups.md).

**Primary group**
: Each user can have **at most one**. Provides the base configuration: home directory, filesystem, quotas, default bandwidth. Group values apply when the corresponding user value is empty or zero.

**Secondary group**
: Any number per user. Adds **incremental** settings: extra virtual folders, per-path permissions, IP filters, extra allowed patterns. Cannot override the root (`/`) path or the root filesystem.

**Membership group**
: Any number per user. **No settings are inherited** — it is a pure tag, used for event-rule conditions (e.g. "only run this rule for members of group X").

## Storage

**Filesystem (backend / provider)**
: The storage system a user's home (or virtual folder) is backed by: local disk, encrypted local (CryptFs), S3, Azure Blob, Google Cloud Storage, SFTP, FTP, HTTP. Each backend has its own configuration shape. See [Features → Storage backends](features.md#storage-backends).

**Virtual folder**
: A mount point that maps a path inside the user's namespace to any storage backend — even a different backend from the user's home. Enables cross-backend workflows (e.g. S3 bucket exposed under `/archive` while the user's home is on local disk). See [Virtual Folders](virtual-folders.md).

**Folder (top-level resource)**
: The server-wide definition of a virtual folder (name + backend + mapped path). A **virtual folder** on a user or group is a reference to a folder plus the *virtual path* at which it is mounted. One folder can be referenced by many users.

**Quota (dedicated vs shared)**
: A virtual folder's quota can be dedicated (the folder counts its own bytes/files) or shared with the user's own quota (value `-1`). Folders shared across users should always use a **dedicated** quota to keep counters accurate.

**DARE / CryptFs**
: Data-At-Rest-Encryption. A local filesystem variant where every file is transparently encrypted with AEAD. See [DARE](dare.md).

## Access control

**Permission**
: Per-path set of allowed operations (`list`, `upload`, `download`, `delete`, `rename`, etc.). Permissions are defined per virtual path; `/` is mandatory for every user.

**DenyPolicy (hide)**
: A file-pattern filter setting. `default` denies access with a permission error; `hide` denies access AND hides the file from directory listings.

**Share**
: A public or password-protected link that exposes a path (or selected files) to external recipients without requiring an SFTPGo account. Shares have a scope (read / write / read+write), optional password, expiration, IP restrictions, and optional email authentication. See [Shares tutorial](tutorials/shares.md).

## Automation

**Event Manager**
: The subsystem that runs **rules → actions** in response to events. See [Event Manager](eventmanager.md).

**Event rule**
: *When* something happens. Binds a trigger, optional conditions, and an ordered list of actions.

**Event action**
: *What* to do. Reusable unit: send an email or HTTP request, run a command, scan a file, copy/move files, apply retention, etc. Actions are referenced by one or more rules.

**Trigger**
: The event type that fires a rule. Types: *Filesystem*, *Provider*, *Schedule*, *IP Blocked*, *Certificate*, *On-demand*, *Identity Provider login*.

**Placeholder / Template**
: A `{{.VariableName}}` expression that is rendered at runtime with contextual data from the triggering event (e.g. `{{.Name}}`, `{{.VirtualPath}}`, `{{.FileSize}}`). Supports helper functions for formatting. See [Placeholders & Templates](placeholders.md).

**Staged upload** (a.k.a. *execute before file publish*)
: An event action mode where file actions run on the *temporary* uploaded file **before** it becomes visible to clients. Used to scan, sign, or transform files before they are published. See [Execute Before File Publish](execute-before-file-publish.md).

## Authentication

**Data provider**
: The database SFTPGo uses to persist users, groups, admins, folders, shares, API keys, and event rules. Can be SQLite, MySQL, PostgreSQL, CockroachDB, or embedded Bolt/Memory. See [Data provider](data-provider.md).

**External auth / Pre-login / Post-login hook**
: Mechanisms to integrate SFTPGo with an external identity source. *External auth* replaces the built-in check, *pre-login* modifies the user record before auth, *post-login* runs side effects after auth. Prefer [Identity Provider login](oidc.md) + Event Manager for modern deployments.

**API key**
: A non-interactive credential used by scripts and integrations to call the REST API without a password or interactive login.

**IdP / OIDC**
: External Identity Provider integration via OpenID Connect. Covers Microsoft Entra ID, Google, Okta, Auth0, Keycloak, and any compatible OIDC provider. SFTPGo does **not** implement SAML. See [OIDC](oidc.md).

## Protection

**Defender**
: Built-in intrusion-detection that tracks failed logins per IP and blocks abusive sources. See [Defender](defender.md).

**Rate limiting**
: Per-protocol limits on requests-per-second that can escalate to a Defender ban. See [Rate limiting](rate-limiting.md).

**Bandwidth limit**
: Per-user / per-group upload and download throughput caps, expressed in KB/s.

## Plugins and hooks

**Plugin**
: An external process (gRPC) that extends SFTPGo with custom behavior: additional KMS providers, authentication logic, notifications, metadata stores. See [Plugins](plugins.md).

**Hook**
: A command or HTTP endpoint invoked by SFTPGo at a specific event (post-connect, post-disconnect, pre-login, post-login, check-password, pre-download, pre-upload, etc.). Hooks are a lightweight alternative to plugins.

## Interfaces

**WebAdmin**
: The browser interface for administrators to manage the server.

**WebClient**
: The browser interface end-users log into to upload/download files, manage shares, and change their password.

**REST API**
: The HTTP API exposed by the server for automation. The authoritative contract is the OpenAPI spec shipped in `openapi/openapi.yaml` (also published at `https://sftpgo.com/assets/openapi.yaml` for the latest release).
