---
description: "Password hashing and validation in open-source SFTPGo. Supports bcrypt, argon2id, and additional formats for migration. Configurable strength policies."
---

# Supported Password Hashing Algorithms

SFTPGo can verify passwords in several formats and uses, by default, the `bcrypt` algorithm to hash passwords in plain-text before storing them inside the data provider. Each hashing algorithm is identified by a prefix.
Supported hash algorithms:

- bcrypt, prefix `$2a$`
- argon2id, prefix `$argon2id$`
- PBKDF2 sha1, prefix `$pbkdf2-sha1$`
- PBKDF2 sha256, prefix `$pbkdf2-sha256$`
- PBKDF2 sha512, prefix `$pbkdf2-sha512$`
- PBKDF2 sha256 with base64 salt, prefix `$pbkdf2-b64salt-sha256$`
- MD5 crypt, prefix `$1$`
- MD5 crypt APR1, prefix `$apr1$`
- SHA256 crypt, prefix `$5$`
- SHA512 crypt, prefix `$6$`
- MD5 digest, prefix `{MD5}`
- SHA256 digest, prefix `{SHA256}`
- SHA512 digest, prefix `{SHA512}`

An administrator with the `*` permission can set a password with one of these prefixes, for example to import accounts from another system: it is stored as is. For other administrators the save fails with a validation error. Backups and the initial data are restored as they are. A password a user chooses, when changing or resetting it or for a share, and the password an administrator chooses for itself are always hashed, even if they start with one of these prefixes.
When users log in, if their passwords are stored with anything other than the preferred algorithm, SFTPGo will automatically upgrade the algorithm to the preferred one.

A plain-text password cannot be longer than 255 characters, 72 bytes with bcrypt. Imported hashes are not subject to this limit.
