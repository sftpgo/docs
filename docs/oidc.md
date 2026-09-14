---
description: "Integrate SFTPGo with OpenID Connect identity providers: Microsoft Entra ID, Google, Okta, Auth0, Keycloak, and more. Supports role mapping and PKCE."
---

# OpenID Connect

OpenID Connect (OIDC) integration allows users and administrators to log in to the SFTPGo WebAdmin and WebClient interfaces using an external Identity Provider (IdP). SFTPGo maps IdP identities to SFTPGo accounts based on configurable claim fields.

OIDC can be configured from the [WebAdmin UI](#configuration-from-the-webadmin-ui), or per HTTP binding in the configuration file and through environment variables so that different ports serve different IdP configurations. All configuration parameters are documented in the [configuration reference](config-file.md#http-server).

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

### Configuration from the WebAdmin UI

The OpenID Connect section of the WebAdmin Configurations page covers the settings most deployments need and stores them in the data provider. To add the section, set the following environment variable before starting SFTPGo:

```shell
SFTPGO_HOOK__ENABLE_OIDC_UI=1
```

See [environment variables](env-vars.md#variable-sources) for where to define it on your platform. After the restart, open **Server Manager > Configurations > OpenID Connect**, fill in the form and click **Submit**. The submitted configuration URL is verified against the provider's discovery document, and a service restart applies the configuration.

The stored configuration is global: every HTTP binding offers the OpenID login, and the flow completes on the address configured as `redirect_base_url`. Configure the bindings in the configuration file, or through environment variables, when different bindings serve different IdP configurations.

Each form field maps to a configuration parameter:

| Field | Parameter |
| ----- | --------- |
| Config URL | `config_url` |
| Client ID | `client_id` |
| Client Secret | `client_secret` |
| Redirect base URL | `redirect_base_url` |
| Username claim | `username_field` |
| Role claim | `role_field` |
| Admin role values | `role_values` |
| User role values | `user_role_values` |
| Implicit roles | `implicit_roles` |
| Scopes | `scopes` |
| Max age | `max_age` |
| Prompt | `prompt` |
| RP-initiated logout | `rp_initiated_logout` |
| Custom claims | `custom_fields` |
| UserInfo claims | `query_userinfo` |
| Verified email | `require_verified_email` |

Claim fields take the claim name as it appears in the token, for example `preferred_username`. Comma-separated fields — scopes, role values, custom claims — take bare values, for example `admin`.

:information_source: `client_secret_file`, `ui_name`, `debug` and the [security settings](#security-settings) come from the configuration file or the environment variables, also when the rest of the configuration is stored from the UI.

### Required settings

| Parameter | Description |
| ----------- | ------------- |
| `config_url` | Base URL of the Identity Provider. SFTPGo appends `/.well-known/openid-configuration` for discovery. SFTPGo will refuse to start if the URL is unreachable. |
| `client_id` | OAuth2 application/client ID. |
| `client_secret` | OAuth2 application/client secret. Configure it whenever the Identity Provider issues one for the application. Can also be provided via `client_secret_file`. Omit both for PKCE-only authentication — see [PKCE without client secret](#pkce-without-client-secret). |
| `redirect_base_url` | Base URL of your SFTPGo instance (e.g., `https://sftpgo.example.com`). SFTPGo appends `/web/oidc/redirect` automatically. Use the same address your users browse: the login is completed on the browser that started it. |
| `username_field` | ID token claim to map to the SFTPGo username (e.g., `preferred_username`, `email`). |

:information_source: Register `<redirect_base_url>/web/oidc/redirect` with your Identity Provider as an allowed redirect URI. The path includes `web_root` when the WebAdmin and WebClient are served from a sub-path.

### Choosing the username claim

The `username_field` claim is the identity key: SFTPGo grants access to the account matching its value. Choose a claim your Identity Provider guarantees to be **unique and stable** for each identity:

- `sub` is the claim the OpenID Connect specification itself guarantees to be unique within the issuer and never reassigned. Using it means provisioning SFTPGo accounts named after the provider-assigned identifier.
- `preferred_username` works well when the provider enforces its uniqueness: Keycloak maps it to the realm username, Microsoft Entra ID to the User Principal Name. The specification leaves its uniqueness to the provider, so verify your provider's policy.
- `email` works well when the provider guarantees it is unique. Enable [`require_verified_email`](#verified-email) to accept it only when the provider asserts the address is verified.

:warning: A claim value shared by two identities, or reassigned to a new person (e.g., a recycled email address or User Principal Name), grants access to the SFTPGo account mapped to that value. Choose a claim users are unable to set for themselves.

### Role mapping

The WebAdmin and the WebClient have separate login pages, each with its own OpenID link, so the page the identity signs in from decides whether an admin or a user session is requested. The role of the identity must authorize that request. There are two approaches:

**Explicit role mapping** — The IdP includes a role claim in the ID token:

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `role_field` | — | ID token claim containing the SFTPGo role. Supports dot notation for nested claims (e.g., `realm_access.roles`). |
| `role_values` | `admin` | Claim values that map to the SFTPGo admin role. Matching is case-insensitive. |
| `user_role_values` | — | Claim values that map to the SFTPGo user role. If empty, any authenticated identity can attempt to log in to the WebClient. Set this to restrict WebClient access to specific claim values. |

An identity whose role claim matches `role_values` can sign in from the WebAdmin login page, one matching `user_role_values` from the WebClient login page; a request the role does not authorize is refused. The mapped username must then exist as an administrator or as a user respectively. The role claim decides administrative access, so choose one the Identity Provider administrator assigns, such as a realm role or a group membership, and not one the identity can edit in its own profile.

**Implicit role mapping** — The role is determined by which login link the user clicks:

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `implicit_roles` | `false` | When `true`, the `role_field` is ignored. Users clicking the admin login link get the admin role; users clicking the client login link get the user role. |

Implicit roles are useful when the IdP does not provide role claims or when you prefer to keep role assignment entirely within SFTPGo. The authorization then rests on the SFTPGo accounts: every identity that authenticates through the admin login link is given the admin role, and the login succeeds when an admin with the mapped username exists and can log in. Keep the set of admin accounts as the list of who administers the server.

### Optional settings

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `scopes` | `openid, profile, email` | OAuth2 scopes to request. The `openid` scope is mandatory. Add custom scopes if your IdP provides additional claims. |
| `custom_fields` | — | Custom ID token claim fields to pass to the pre-login hook and Event Manager. See [Custom fields](#custom-fields). |
| `max_age` | — | Maximum allowed seconds since the user last actively authenticated. Forces re-authentication if exceeded. Set to `0` to always force re-authentication. If empty, the IdP's default policy applies. |
| `prompt` | — | Controls the IdP's authentication/consent behavior. Common values: `none`, `login`, `consent`, `select_account`. Space-delimited. |
| `rp_initiated_logout` | `false` | Redirects the browser to the IdP's end session endpoint on logout, so the single sign-on session is terminated (OpenID Connect RP-Initiated Logout). The SFTPGo login page is sent as `post_logout_redirect_uri` and must be registered with your IdP as an allowed post-logout redirect URI. |
| `query_userinfo` | `false` | Queries the IdP's UserInfo endpoint after authentication and reads user claims from both sources. See [UserInfo claims](#userinfo-claims). |
| `require_verified_email` | `false` | Accepts only identities whose email address is verified by the IdP. See [Verified email](#verified-email). |
| `ui_name` | `OpenID` | Label displayed on the login button (e.g., "Sign in with *ui_name*"). |
| `debug` | `false` | Log received ID tokens at debug level. Useful for troubleshooting claim mapping. |

### Security settings

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `disabled_security_features` | `0` | Set to `1` to disable PKCE (Proof Key for Code Exchange). PKCE is enabled by default; keep it enabled when the client secret is omitted, since it then secures the exchange on its own. See [PKCE without client secret](#pkce-without-client-secret). |
| `insecure_skip_signature_check` | `false` | :warning: Accepts the ID token without validating its signature. Supported for providers that sign ID tokens with the `none` algorithm. SFTPGo reads the ID token from the provider's token endpoint over TLS, which is the only integrity guarantee left while this setting is enabled. |
| `insecure_issuer_url` | `false` | Allow the issuer URL reported by the provider to differ from the discovery URL. Required for off-spec providers like Azure B2C. |
| `issuer_url` | — | Explicit issuer URL for token verification, applied when `insecure_issuer_url` is enabled. |

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

## UserInfo claims

By default, all claims (`username_field`, `role_field`, `custom_fields`) are read from the verified ID token. Some Identity Providers return profile claims from the UserInfo endpoint and require extra configuration to include them in the ID token.

With `query_userinfo` enabled, SFTPGo queries the provider's UserInfo endpoint after each authentication and reads the claims from both sources:

- Non-empty ID token claims take precedence over UserInfo claims with the same name, so the UserInfo response fills in missing claims. A claim set to null, to an empty string or to an empty list counts as missing in both sources: if the ID token returns an empty `username_field` or `role_field`, the value from the UserInfo response is used. Enable `query_userinfo` when the UserInfo claims are as authoritative as the ID token ones for these two fields.
- The UserInfo subject must match the ID token subject, as required by the OIDC specification. Authentication fails on mismatch or when the UserInfo request fails.
- `sid`, `auth_time` and `nonce` are read from the ID token only: SFTPGo uses them for session tracking and for the `max_age` check.

The provider must advertise a UserInfo endpoint in its discovery document; this is validated at startup.

:information_source: Microsoft Entra ID returns a fixed set of claims from the UserInfo endpoint and recommends reading claims from the ID token, which also saves a network round-trip per login. Enable `query_userinfo` when your provider returns the claims you need from the UserInfo endpoint only.

## Verified email

With `require_verified_email` enabled, the login is allowed only if the `email_verified` claim is set to `true`. The claim is read from the verified ID token or, when `query_userinfo` is enabled, from the UserInfo response, so a provider that returns `email_verified` from the UserInfo endpoint only is supported.

:warning: Verify that your provider returns `email_verified` before enabling this setting: with a provider that omits the claim, every login is refused.

The `email_verified` claim belongs to the `email` scope, which governs both sources: the UserInfo endpoint returns the claims authorized by the scopes granted to the access token, so dropping `email` from `scopes` normally removes the claim from the ID token and from the UserInfo response alike. Keep that scope in `scopes`, or configure the IdP to return the claim regardless of the requested scopes, as Keycloak does with its default client scopes.

:information_source: The claim is checked at login. A session already established keeps working until it expires, so a mail address that becomes unverified at the IdP is enforced on the next login.

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

SFTPGo completes the authorization code exchange from the server: it stores the client secret encrypted in the data provider, or reads it from `client_secret_file`. Configure `client_secret` whenever the Identity Provider issues one for the application.

Omit `client_secret` and `client_secret_file` when the Identity Provider registers the application without a secret: some providers reserve secrets to specific client types, some organization policies assign the application a type that carries none, and an IdP owner may keep its secrets from the party that operates the SFTPGo instance. PKCE (Proof Key for Code Exchange) then secures the exchange on its own, so keep it enabled — it is the default, and `disabled_security_features` turns it off.

A configuration without a client secret:

```shell
SFTPGO_HTTPD__BINDINGS__0__OIDC__CLIENT_ID="sftpgo-public-client"
SFTPGO_HTTPD__BINDINGS__0__OIDC__CONFIG_URL="https://idp.example.com/realms/sftpgo"
SFTPGO_HTTPD__BINDINGS__0__OIDC__REDIRECT_BASE_URL="https://sftpgo.example.com"
SFTPGO_HTTPD__BINDINGS__0__OIDC__USERNAME_FIELD="preferred_username"
SFTPGO_HTTPD__BINDINGS__0__OIDC__ROLE_FIELD="sftpgo_role"
```

The Identity Provider must identify the client as public too. In Keycloak, set the client's **Access Type** to `public` (or **Client authentication** to `Off` in newer versions). In Microsoft Entra ID the platform of the registered redirect URI decides it, see [public client without a client secret](#public-client-without-a-client-secret). Other providers have similar settings — consult your IdP documentation.

:information_source: PKCE binds the authorization code to the login that started it, and the client secret identifies SFTPGo at the token endpoint. A public client registration applies to the whole application, so keep it narrow: one registered redirect URI, the one SFTPGo serves, and the flows that carry no redirect URI, device code among them, left disabled. The client ID travels in every authorization request, and with those two in place it stays usable for this login alone.

## Troubleshooting

The login page reports the reason a login was refused, and the SFTPGo log carries the details. Set `debug` to `true` to log the received ID token and see the claims your provider returns.

| Message on the login page | Cause |
| ------------------------- | ----- |
| Failed to exchange OpenID token | The Identity Provider refused the token request, or the UserInfo query failed when [`query_userinfo`](#userinfo-claims) is enabled. The log carries the provider's own message. |
| Invalid OpenID token | The claim configured as `username_field` carries no value, or the identity authenticated longer ago than `max_age` allows. |
| Incorrect OpenID role | The identity's role claim authorizes the other login page, or its value is missing from `role_values`/`user_role_values`. See [Role mapping](#role-mapping). |
| Failed to get user associated with OpenID token | The mapped username has no SFTPGo account yet. See [Account provisioning](#account-provisioning). |
| The email address is not verified | `require_verified_email` is enabled and the provider asserts no verified address. See [Verified email](#verified-email). |

An invalid redirect URI is reported by the Identity Provider, before the browser returns to SFTPGo: check the URI registered with the provider against the one built from [`redirect_base_url`](#required-settings).

When the provider asks for client credentials and the configuration omits the client secret, review the client type the provider expects for the registered redirect URI: see [PKCE without client secret](#pkce-without-client-secret).

When the username claim carries no value, the log names the configured claim and lists the claims the token carries:

```
username field "preferred_username" not found, empty or not a string, claims fields: [sub email_verified name preferred_username given_name family_name email]
```

Compare the configured claim with that list: it must be one of those names. A claim your provider returns from the UserInfo endpoint alone is read when [`query_userinfo`](#userinfo-claims) is enabled, and a claim the provider returns to a specific scope needs that scope in `scopes`.

## Provider-specific notes

### Microsoft Entra ID (Azure AD)

Standard configuration works for most setups. Entra ID signs ID tokens with RS256 and publishes its signing keys in the discovery document, so set `config_url` to the tenant's v2.0 endpoint, `https://login.microsoftonline.com/<tenant-id>/v2.0`.

If signature validation fails, verify that `config_url` returns the discovery document of the tenant that issues the tokens: the keys used for validation come from that document. `insecure_skip_signature_check` covers providers that sign ID tokens with the `none` algorithm; a provider that publishes signing keys works with the default settings.

#### Public client without a client secret

Entra ID issues client secrets for applications registered under the **Web** platform, so a client secret covers this provider. To register the application as a public client instead, and authenticate with [PKCE alone](#pkce-without-client-secret), the platform the redirect URI is registered under is what decides the client type:

1. Open **App registrations** => your application => **Authentication**.
2. Remove `<redirect_base_url>/web/oidc/redirect` from the **Web** platform.
3. Add the same URI under **Mobile and desktop applications** => **Custom redirect URIs**.
4. Leave **Allow public client flows** set to **No**.

While the URI stays registered under **Web**, Entra ID asks for the client secret and reports `AADSTS7000218`. The **Allow public client flows** setting governs the flows that carry no redirect URI, such as device code: this login carries one, so it follows the platform of the redirect URI and works with that setting left at **No**.

:warning: Keep the URI under **Mobile and desktop applications** rather than **Single-page application**: the SPA platform serves code redemption from the browser, and SFTPGo completes the exchange from the server.

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
2. In **Realm Settings** => **Login**, adjust the "Require SSL" setting for your environment. Ensure "Unmanaged Attributes" are allowed if you plan to use custom attributes.
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

The ID token must contain the `username_field` claim, and the mapped username must exist in SFTPGo (or be provisioned automatically via the Event Manager or pre-login hook). Signing in from the WebAdmin login page requires the `role_field` claim with the value `admin`; the WebClient login page accepts any authenticated identity while `user_role_values` is empty.

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
