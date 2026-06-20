---
description: "Password hashing and validation in SFTPGo. Supports bcrypt, argon2id, and 12 additional formats for migration. Configurable strength policies."
---

# Password Hashing

SFTPGo hashes passwords before storing them in the data provider. By default, the `bcrypt` algorithm is used.

## Preferred algorithm

The preferred hashing algorithm can be configured in the `data_provider.password_hashing` section of the [configuration file](config-file.md#password_hashing). Supported algorithms for new passwords:

- **bcrypt** (default) — prefix `$2a$`. Configurable cost parameter (default: 10).
- **argon2id** — prefix `$argon2id$`. Configurable memory, iterations, and parallelism.
- **pbkdf2-sha256** — prefix `$pbkdf2-b64salt-sha256$`. PBKDF2 with HMAC-SHA256 and a random salt. Configurable iteration count (default: 600000) and salt length (default: 16 bytes, valid range 16--64).

When users log in, if their password is stored with a different algorithm than the preferred one, SFTPGo automatically re-hashes it with the preferred algorithm. This upgrade happens transparently; the user does not need to change their password. The same transparent upgrade applies to administrator and share passwords at their next successful verification. API keys keep the format they were generated with: generate a new key to adopt the preferred algorithm.

## Supported formats

In addition to the preferred algorithm, SFTPGo can verify passwords stored in many other formats. This allows migrating users from other systems (such as OpenSSH, OpenLDAP, Apache, or other SFTP/FTP servers) by importing their existing password hashes directly into SFTPGo. Users can log in immediately with their current passwords, and at first login SFTPGo will automatically re-hash each password using the preferred algorithm.

| Format | Prefix |
| ------ | ------ |
| bcrypt | `$2a$` |
| argon2id | `$argon2id$` |
| PBKDF2-SHA1 | `$pbkdf2-sha1$` |
| PBKDF2-SHA256 | `$pbkdf2-sha256$` |
| PBKDF2-SHA512 | `$pbkdf2-sha512$` |
| PBKDF2-SHA256 (base64 salt) | `$pbkdf2-b64salt-sha256$` |
| MD5-Crypt | `$1$` |
| MD5-Crypt (APR1) | `$apr1$` |
| SHA256-Crypt | `$5$` |
| SHA512-Crypt | `$6$` |
| yescrypt | `$y$` |
| MD5 digest | `{MD5}` |
| SHA256 digest | `{SHA256}` |
| SHA512 digest | `{SHA512}` |

If you set a password with one of these prefixes, it will not be re-hashed — SFTPGo will store it as-is and verify it using the corresponding algorithm.

## Password caching

Verifying bcrypt, argon2id, and pbkdf2-sha256 hashes is computationally expensive. SFTPGo caches verified passwords in memory to reduce CPU usage on repeated logins. Caching is enabled by default and can be disabled via the `data_provider.password_caching` [configuration option](config-file.md).

## Password validation

Password strength rules are validated every time a password is set or changed — when an administrator creates or updates a user, when the user changes their own password, on password reset, and when a [share](tutorials/shares.md) is created or updated with a password.

Available rule types:

- **Entropy-based** — evaluates overall cryptographic strength (`min_entropy` at system level, `password_strength` on users and groups). This is the recommended approach.
- **Character rules** (`password_policy`) — minimum length, uppercase, lowercase, digits, and special characters.

### Admin passwords

Admin accounts do not have per-admin password fields. The `data_provider.password_validation.admins.*` values in the [configuration file](config-file.md#password_validation) are the **enforced minimum** — every admin password must satisfy them, with no way to override them per admin.

### Protocol user and share passwords

For protocol users (and passwords set on [shares](tutorials/shares.md)), rules can be defined at three levels, with the following precedence (highest first):

1. **Per user** — `password_strength` and `password_policy` on the user record (`Filters` section).
2. **Primary group** — the same fields on the primary group's user settings. Each field is taken from the primary group when the user's corresponding field is `0`/empty. Secondary groups are **not** considered for password policy.
3. **System default** — `data_provider.password_validation.users.*` in the configuration file. Each field is used when the effective user value (after the primary-group merge) is `0`/empty.

:warning: **For protocol users, the system-level values are defaults, not hard floors.** Any user or primary group can override them — including to a **weaker** value. This is intentional: administrators may need to configure less-strict requirements for service accounts or legacy integrations. If your deployment requires a non-bypassable minimum for protocol users, restrict which administrators can set `password_strength` / `password_policy` on users and groups.

:warning: **All rules are disabled by default (value `0`), meaning any password is accepted.** It is strongly recommended to configure at least a system-level default so admins and users without per-entity settings are still protected.
