---
description: "Dynamically create or modify SFTPGo users before login using external programs or HTTP hooks for just-in-time provisioning."
---

# Dynamic user creation or modification

Dynamic user creation or modification is supported via an external program or an HTTP URL that can be invoked just before the user login.
To enable dynamic user modification, you must set the absolute path of your program or an HTTP URL using the `pre_login_hook` key in your configuration file.

The external program can read the following environment variables to get info about the user trying to login:

- `SFTPGO_LOGIND_USER`, it contains the user trying to login serialized as JSON. A JSON serialized user id equal to zero means the user does not exist inside SFTPGo
- `SFTPGO_LOGIND_METHOD`, possible values are: `password`, `publickey`, `keyboard-interactive`, `TLSCertificate`, `IDP` (external identity provider) or empty if the hook is executed after receiving the FTP `USER` command. For [multi-step authentication](ssh.md#multi-step-authentication) the hook is invoked once per step with the single-step factor (`password`, `publickey` or `keyboard-interactive`); the combined name (e.g. `publickey+password`) is not exposed to the pre-login hook.
- `SFTPGO_LOGIND_IP`, ip address of the user trying to login
- `SFTPGO_LOGIND_PROTOCOL`, possible values are `SSH`, `FTP`, `DAV`, `HTTP`, `OIDC` (OpenID Connect)

The program must write, on its standard output:

- an empty string (or no response at all) if the user should not be created/updated
- or the SFTPGo user, JSON serialized, if you want to create or update the given user

If the hook is an HTTP URL then it will be invoked as HTTP POST. The login method, the used protocol and the ip address of the user trying to login are added to the query string, for example `<http_url>?login_method=password&ip=1.2.3.4&protocol=SSH`.
The request body will contain the user trying to login serialized as JSON. If no modification is needed the HTTP response code must be 204, otherwise the response code must be 200 and the response body a valid SFTPGo user serialized as JSON.

Actions defined for user's updates will not be executed in this case and an already logged in user with the same username will not be disconnected, you have to handle these things yourself.

The program hook must finish within 30 seconds, the HTTP hook will use the global configuration for HTTP clients.

If an error happens while executing the hook then login will be denied.

:information_source: the pre-login hook is a trusted component, see [Trust model](external-auth.md#trust-model).

"Dynamic user creation or modification" and "External Authentication" are mutually exclusive. With "External Authentication" the external program receives the credentials of the user trying to login, for example the cleartext password, validates them and returns an already authenticated user. With "Dynamic user creation or modification" the pre-login program receives the user stored inside the data provider, including the hashed password if any, and can modify it; SFTPGo then checks the credentials of the user trying to login against the returned user.

The pre-login hook is executed even if an external authentication hook is defined in the following cases:

- for SFTPGo users (not admins) authenticating with an external identity provider such as OpenID Connect, the hook runs after a successful authentication against the identity provider, so that you can create or update the SFTPGo user matching the authenticated one.
- if FTP allows both encrypted and plain text sessions, the hook also runs after the FTP `USER` command of a session that has not switched to TLS yet. If the returned user has `ftp_security` set to `1`, the client is told that TLS is required and the session is closed before it can send a password.

You can disable the hook on a per-user basis.

You can also instruct SFTPGo to skip the pre-login hook for recently active users by setting `pre_login_cache_time` (seconds) on the user object. While the user's last login is within the configured window the hook is bypassed. This is useful for protocols like WebDAV where every HTTP request re-authenticates and the hook would otherwise fire on every request. Set to `0` (default) to always run the hook.

The structure for SFTPGo users can be found within the [OpenAPI schema](https://sftpgo.com/rest-api){:target="_blank"}.

## The returned user

The returned user replaces the stored one: groups, virtual folders, permissions, filters, filesystem and role are the ones the response carries, and a field the response omits is cleared. SFTPGo keeps only the state the hook cannot know: the id, the quota and transfer counters, the last login and password change, the first upload and download, the two-factor configuration and the recovery codes. Start from the user received in input and return it with your changes to keep everything else as it is.

The `role` follows the same rule. With [resource isolation](roles.md#resource-isolation) enabled, a response that drops the role of an account referencing groups or virtual folders of its tenant is refused and the login is denied; an account with no groups or folders is saved without a role.

## Auto-created virtual folders

The virtual folders named in the response must already exist, as for an account saved through the REST API. With a SQL data provider (PostgreSQL, MySQL, SQLite, CockroachDB) you can set the `SFTPGO_HOOK__AUTO_FOLDERS` environment variable to `1` to create them with the account: each entry of `virtual_folders` can carry the folder definition, `name`, `mapped_path`, `filesystem` and `description`. A folder that does not exist is created with those fields; one that exists is updated with them. A folder is created with the role of the account when the response names none, and a response naming a different role is refused. The [storage allowlist](roles.md#storage-allowlist) of the account's role applies to the created folders.

The role of an existing folder is left as it is, so with [resource isolation](roles.md#resource-isolation) enabled a folder that belongs to another role cannot be attached to the account and the save fails.

:information_source: the same applies to the [external authentication](external-auth.md) hook and to the authentication plugins.

## Placeholder users

The pre-login hook is invoked for every username, including the ones that do not exist inside SFTPGo. It can return a user for an unknown username too: SFTPGo creates or updates the returned account and then checks the credentials against it, so the login of an unknown username is refused like a login with wrong credentials.

To handle every unknown username in the same way, return the same disabled placeholder account, with `status` set to `0`, for all of them: SFTPGo matches the returned username against the data provider, so a single account is created the first time and updated on the following logins, and logging in with the placeholder username is denied as well.

:information_source: the same pattern is available for [external authentication](external-auth.md) and for authentication plugins: the account they return is created or updated in the same way.
