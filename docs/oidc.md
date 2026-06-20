---
description: "Integrate SFTPGo with OpenID Connect identity providers: Microsoft Entra ID, Google, Okta, Auth0, Keycloak, and more. Supports role mapping and PKCE."
---

# OpenID Connect

OpenID Connect (OIDC) integration allows users and administrators to log in to the SFTPGo WebAdmin and WebClient interfaces using an external Identity Provider (IdP). SFTPGo maps IdP identities to SFTPGo accounts based on configurable claim fields.

OIDC is configured per HTTP binding — you can have different IdP configurations on different ports if needed. All configuration parameters are documented in the [configuration reference](config-file.md#http-server).

## How it works

1. The user clicks "Sign in with OpenID" (or the configured label) on the SFTPGo login page.
2. SFTPGo redirects to the Identity Provider's authorization endpoint.
3. The user authenticates with the IdP.
4. The IdP redirects back to SFTPGo with an authorization code.
5. SFTPGo exchanges the code for tokens, validates the ID token (signature, nonce, expiration), and extracts the configured claims.
6. If an [Event Manager](eventmanager.md) rule with an IdP login trigger is configured, it executes (e.g., to create or update the account automatically).
7. SFTPGo looks up the user or admin by the mapped username and establishes a session.

SFTPGo uses [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html){:target="_blank"} — it appends `/.well-known/openid-configuration` to the configured `config_url` to automatically discover the provider's endpoints.

## Configuration

### Required settings

| Parameter | Description |
| ----------- | ------------- |
| `config_url` | Base URL of the Identity Provider. SFTPGo appends `/.well-known/openid-configuration` for discovery. SFTPGo will refuse to start if the URL is unreachable. |
| `client_id` | OAuth2 application/client ID. |
| `client_secret` | OAuth2 application/client secret. Can also be provided via `client_secret_file`. Optional when using PKCE-only authentication — see [PKCE without client secret](#pkce-without-client-secret). |
| `redirect_base_url` | Base URL of your SFTPGo instance (e.g., `https://sftpgo.example.com`). SFTPGo appends `/web/oidc/redirect` automatically. |
| `username_field` | ID token claim to map to the SFTPGo username (e.g., `preferred_username`, `email`). |

### Role mapping

SFTPGo needs to determine whether an authenticated user should access the WebAdmin (as an admin) or the WebClient (as a user). There are two approaches:

**Explicit role mapping** — The IdP includes a role claim in the ID token:

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `role_field` | — | ID token claim containing the SFTPGo role. Supports dot notation for nested claims (e.g., `realm_access.roles`). |
| `role_values` | `admin` | Claim values that map to the SFTPGo admin role. Matching is case-insensitive. |
| `user_role_values` | — | Claim values that map to the SFTPGo user role. If empty, any authenticated identity can attempt to log in to the WebClient. Set this to restrict WebClient access to specific claim values. |

**Implicit role mapping** — The role is determined by which login link the user clicks:

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `implicit_roles` | `false` | When `true`, the `role_field` is ignored. Users clicking the admin login link get the admin role; users clicking the client login link get the user role. |

Implicit roles are useful when the IdP does not provide role claims or when you prefer to keep role assignment entirely within SFTPGo.

### Optional settings

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `scopes` | `openid, profile, email` | OAuth2 scopes to request. The `openid` scope is mandatory. Add custom scopes if your IdP provides additional claims. |
| `custom_fields` | — | Custom ID token claim fields to pass to the pre-login hook and Event Manager. See [Custom fields](#custom-fields). |
| `max_age` | — | Maximum allowed seconds since the user last actively authenticated. Forces re-authentication if exceeded. Set to `0` to always force re-authentication. If empty, the IdP's default policy applies. |
| `prompt` | — | Controls the IdP's authentication/consent behavior. Common values: `none`, `login`, `consent`, `select_account`. Space-delimited. |
| `ui_name` | `OpenID` | Label displayed on the login button (e.g., "Sign in with *ui_name*"). |
| `debug` | `false` | Log received ID tokens at debug level. Useful for troubleshooting claim mapping. |

### Security settings

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `disabled_security_features` | `0` | Set to `1` to disable PKCE (Proof Key for Code Exchange). PKCE is enabled by default and recommended. |
| `insecure_skip_signature_check` | `false` | :warning: Skip JWT signature validation. Only for providers that use the `none` signing algorithm (e.g., some Azure configurations). |
| `insecure_issuer_url` | `false` | Allow the issuer URL reported by the provider to differ from the discovery URL. Required for off-spec providers like Azure B2C. |
| `issuer_url` | — | Explicit issuer URL for token verification. Only applied when `insecure_issuer_url` is enabled. |

## Account provisioning

The mapped username must correspond to an existing SFTPGo user or admin. If the account does not exist at login time, authentication fails. There are two ways to automatically provision accounts:

### Event Manager (recommended)

Create an Event Manager rule with an **Identity Provider login** trigger and an **Identity Provider account check** action. This action uses Go templates to generate the user or admin object from the ID token claims, and can either create the account on first login or update it on every login.

Custom IdP fields are available in templates as `{{.IDPFields.fieldname}}`. See the [Placeholders & Templates](placeholders.md) reference.

### Pre-login hook

Alternatively, use a [pre-login hook](dynamic-user-mod.md) to create or modify users dynamically. Custom fields configured in `custom_fields` are passed to the hook in the `oidc_custom_fields` field of the JSON payload.

## Custom fields

The `custom_fields` configuration lists additional ID token claims to extract and make available to account provisioning:

1. Add the claim name to `custom_fields` in the OIDC configuration.
2. If needed, add a custom scope to `scopes` so the IdP includes the claim in the ID token.
3. Access the field in Event Manager templates as `{{.IDPFields.fieldname}}` or in the pre-login hook via the `oidc_custom_fields` JSON field.

All custom fields are passed with their original types as defined by the Identity Provider.

## Security

SFTPGo implements the following security measures for OIDC:

- **PKCE** (Proof Key for Code Exchange) is enabled by default, preventing authorization code interception attacks.
- **Nonce validation** — each authentication request includes a unique nonce that must match the ID token.
- **State parameter** — CSRF protection for the OAuth2 flow.
- **ID token signature verification** — tokens are cryptographically verified against the provider's public keys.
- **Auth time validation** — when `max_age` is configured, the `auth_time` claim is checked with a 60-second clock skew tolerance.
- **Session cookies** — set with `HttpOnly`, `Secure`, and `SameSite=Lax` attributes.

## OIDC and local credentials

OpenID Connect authenticates the **web session** only. The mapped account is a regular SFTPGo user or admin, so it keeps its own SFTPGo credentials (password, public keys) and its own set of allowed protocols. When those are present and permitted, they are independent authentication paths that work without OIDC:

- A user with a password and/or public keys can connect over SSH/SFTP, FTP and WebDAV when the corresponding protocol and login method are allowed.
- A user with a password can also sign in to the WebClient through the password login form when the `password` login method is allowed for HTTP.

:warning: These credentials are stored in SFTPGo and follow the SFTPGo account lifecycle, not the Identity Provider's: disabling, locking or deleting the account in the IdP does not revoke them. The user keeps working over the allowed protocols until the SFTPGo account is updated.

### Restrict an account to OIDC only

Use the per-user (or per-group) login and protocol filters:

- **WebClient via OIDC only** — keep the `HTTP` protocol allowed and add `password` to `denied_login_methods`. OIDC login keeps working; the password login form is rejected.

    :warning: Do not add `HTTP` to `denied_protocols` for this purpose: OIDC web login runs over HTTP, so denying the protocol disables OIDC too.

- **No direct SSH/SFTP, FTP or WebDAV** — add `SSH`, `FTP` and `DAV` to `denied_protocols`, or add `password`, `publickey` and `keyboard-interactive` to `denied_login_methods`.

Applying these as group settings enforces the policy across many accounts at once.

### Revoke access when the IdP account is invalidated

SFTPGo does not poll the Identity Provider, so revoking an account that already has local credentials requires one of:

- Disable (set the status to inactive) or delete the account through the [REST API](rest-api.md), driven by your IdP's user-lifecycle automation.
- Gate access at login with an **Identity Provider login** Event Manager rule or a [pre-login hook](dynamic-user-mod.md). These run at OIDC login time, so combine them with the restrictions above when the account also allows SSH/FTP/WebDAV, otherwise direct protocol logins bypass the check.

## PKCE without client secret

SFTPGo supports PKCE-only authentication, where the `client_secret` is omitted entirely. This is useful when the Identity Provider is configured with a **public client** (no client secret), which is common in scenarios where storing a secret securely is not practical or when the IdP policy mandates public clients.

When the client secret is not configured, SFTPGo uses PKCE (Proof Key for Code Exchange) exclusively to secure the authorization code exchange. PKCE must remain enabled (the default) — if you disable it via `disabled_security_features`, PKCE-only authentication will not work.

To configure PKCE-only authentication, simply omit the `client_secret` and `client_secret_file` parameters:

```shell
SFTPGO_HTTPD__BINDINGS__0__OIDC__CLIENT_ID="sftpgo-public-client"
SFTPGO_HTTPD__BINDINGS__0__OIDC__CONFIG_URL="https://idp.example.com/realms/sftpgo"
SFTPGO_HTTPD__BINDINGS__0__OIDC__REDIRECT_BASE_URL="https://sftpgo.example.com"
SFTPGO_HTTPD__BINDINGS__0__OIDC__USERNAME_FIELD="preferred_username"
SFTPGO_HTTPD__BINDINGS__0__OIDC__ROLE_FIELD="sftpgo_role"
```

:information_source: In Keycloak, set the client's **Access Type** to `public` (or **Client authentication** to `Off` in newer versions). In Azure AD, register the application as a public client. Other providers have similar settings — consult your IdP documentation.

## Provider-specific notes

### Microsoft Entra ID (Azure AD)

Standard configuration works for most setups. If you encounter signature validation errors, the provider may be using the `none` signing algorithm — set `insecure_skip_signature_check` to `true`.

### Azure AD B2C

Azure B2C uses a discovery URL that differs from the issuer URL in the ID token. Enable `insecure_issuer_url` and set `issuer_url` to the value your B2C tenant reports:

```shell
SFTPGO_HTTPD__BINDINGS__0__OIDC__INSECURE_ISSUER_URL=true
SFTPGO_HTTPD__BINDINGS__0__OIDC__ISSUER_URL="https://your-tenant.b2clogin.com/your-tenant-id/v2.0/"
```

### Google, Okta, Auth0, OneLogin, Amazon Cognito, Ping Identity, JumpCloud

Standard OIDC configuration. No special settings required.

### Keycloak

See the [example configuration](#example-keycloak-setup) below.

## Example: Keycloak setup

This example shows a basic integration with [Keycloak](https://www.keycloak.org/){:target="_blank"}. Other OpenID Connect providers follow a similar pattern.

### Keycloak preparation

1. Create a realm named `sftpgo`.
2. In **Realm Settings** → **Login**, adjust the "Require SSL" setting for your environment. Ensure "Unmanaged Attributes" are allowed if you plan to use custom attributes.
3. Create a client named `sftpgo-client` with **Access Type** set to `confidential`.
4. Set a valid redirect URI — for example, `http://192.168.1.50:8080/*` if SFTPGo runs at that address.
5. In the client's **Mappers** settings, ensure that the username and role are included in the ID token. For example, map the user attribute `sftpgo_role` as a JSON string to the ID token, and `username` as `preferred_username`.
6. For users who should have admin access, add a custom attribute with key `sftpgo_role` and value `admin`.

### SFTPGo configuration

Using environment variables (recommended):

```shell
SFTPGO_HTTPD__BINDINGS__0__OIDC__CLIENT_ID="sftpgo-client"
SFTPGO_HTTPD__BINDINGS__0__OIDC__CLIENT_SECRET="jRsmE0SWnuZjP7djBqNq0mrf8QN77j2c"
SFTPGO_HTTPD__BINDINGS__0__OIDC__CONFIG_URL="http://192.168.1.12:8086/auth/realms/sftpgo"
SFTPGO_HTTPD__BINDINGS__0__OIDC__REDIRECT_BASE_URL="http://192.168.1.50:8080"
SFTPGO_HTTPD__BINDINGS__0__OIDC__USERNAME_FIELD="preferred_username"
SFTPGO_HTTPD__BINDINGS__0__OIDC__ROLE_FIELD="sftpgo_role"
```

Or equivalently in the configuration file:

```json
"oidc": {
  "client_id": "sftpgo-client",
  "client_secret": "jRsmE0SWnuZjP7djBqNq0mrf8QN77j2c",
  "config_url": "http://192.168.1.12:8086/auth/realms/sftpgo",
  "redirect_base_url": "http://192.168.1.50:8080",
  "username_field": "preferred_username",
  "role_field": "sftpgo_role"
}
```

### How it works

From the SFTPGo login page, click "Sign in with OpenID". You are redirected to Keycloak's login page. After successful authentication, Keycloak redirects back to SFTPGo.

The ID token must contain the `username_field` claim. The mapped username must exist in SFTPGo (or be provisioned automatically via the Event Manager or pre-login hook). If the token contains the `role_field` claim with value `admin`, the user is directed to the WebAdmin; otherwise, to the WebClient.

Example ID token for an admin:

```json
{
    "preferred_username": "root",
    "sftpgo_role": "admin",
    "email": "root@example.com"
}
```

Example ID token for a regular user (no role claim needed):

```json
{
    "preferred_username": "user1",
    "email": "user1@example.com"
}
```

If you don't want to manage roles in the IdP, set `implicit_roles` to `true` — the role will be determined by which login link the user clicks.
