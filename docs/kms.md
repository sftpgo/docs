---
description: "Configure Key Management Services in SFTPGo for encrypting stored secrets. Supports local encryption and cloud KMS providers."
---

# Key Management Services

SFTPGo stores sensitive data such as Cloud account credentials or passphrases to derive per-object encryption keys. These data are stored as ciphertext and only loaded to RAM in plaintext when needed.

## Supported Services for encryption and decryption

The `secrets` section of the `kms` configuration allows to configure how to encrypt and decrypt sensitive data. The following configuration parameters are available:

- `url` defines the URI to the KMS service. If empty, the default `AES-256-GCM` provider is used.
- `master_key` defines the active master encryption key as a string, used to encrypt and as the first key tried on decryption.
- `master_key_path` defines the absolute path to a file containing the active master encryption key. This could be, for example, a docker secret or a file protected with filesystem level permissions. When set it takes precedence over `master_key`.
- `additional_master_keys` is a list of decrypt-only master keys, tried in order only when the active key fails to decrypt a secret. They never encrypt and let the data store hold secrets written under more than one key during a key rotation. See the [master key ring](config-file.md#master-key-ring).
- `additional_master_keys_path` defines the absolute path to a file holding additional decrypt-only master keys, one key per line, blank lines ignored. When set it takes precedence over `additional_master_keys`.

### Default provider (AES-256-GCM)

If the `url` is empty (or set to `aes256gcm://`) SFTPGo encrypts secrets using `AES-256-GCM` authenticated encryption.

For each secret a random salt is generated and the per-object encryption key is derived from it in the following way:

1. a master key is provided: the encryption key is derived from the master key and the per-secret salt using the HMAC-based Extract-and-Expand Key Derivation Function (HKDF) as defined in [RFC 5869](http://tools.ietf.org/html/rfc5869){:target="_blank"}. The master key is never stored alongside the ciphertext, so a compromise of the data store alone is not sufficient to decrypt the secrets.
2. no master key is provided: the encryption key is derived from the per-secret salt alone. This is the default configuration.

Without a master key the secrets are recoverable from a copy of the data store alone; set a master key to keep credentials confidential against a database or backup compromise. This is especially relevant for high-value, reusable credentials such as cloud account access keys.

### Local provider (NaCl secretbox)

You can activate the local provider by setting `local://` as `url`. Internally it uses the [NaCl secret box](https://pkg.go.dev/golang.org/x/crypto/nacl/secretbox){:target="_blank"} algorithm and the same key derivation scheme (HKDF, optional master key) described above. This was the default in previous SFTPGo versions; secrets already encrypted with it are transparently decrypted regardless of the currently configured provider.

### Cloud providers

The following cloud KMS providers are supported via the KMS plugin:

- Google Cloud KMS
- AWS KMS
- Azure Key Vault
- HashiCorp Vault
- Oracle Key Vault

All cloud providers that offer managed identity (Google Cloud, AWS, Azure, Oracle) are supported through their respective default credential chains.

See [Cloud KMS Providers](plugins/kms-providers.md) for detailed configuration instructions.

## Re-encrypting stored secrets

The master key is global and is **not** stored with the secret: every secret is decrypted with the master key currently configured. Changing the master key in the configuration therefore makes the secrets encrypted with the previous key undecryptable, and the users that need them in plain text can no longer log in. The provider, instead, is recorded per secret (by its status), so secrets encrypted by one provider keep being decrypted by it even after the configured provider changes.

The `convertsecrets` command re-encrypts every stored secret onto a new master key and/or provider in a single pass, so you can rotate the master key or migrate provider without losing access to the data. To rotate the master key (for example to replace a weak one), keep the current key in the configuration so the existing secrets can still be decrypted, run the command with the new key, and only then update the configuration to the new key. To migrate to a different provider, run it with the target provider: a builtin one (`local` or `aes256gcm`) or the full provider URL of an external KMS whose plugin is configured. See [Re-encrypting the stored secrets](cli.md#re-encrypting-the-stored-secrets) for the full usage. On a networked SQL database (PostgreSQL, MySQL/MariaDB, CockroachDB), whether a single instance or a cluster, you can rotate the master key with no downtime by configuring a [master key ring](config-file.md#master-key-ring) so the running instances decrypt both keys while the secrets are migrated.

:information_source: The KMS master key wraps the secrets stored in the data provider — cloud credentials, two-factor secrets, and the **CryptFs passphrase**, which is itself stored as a KMS secret. `convertsecrets` re-wraps these secrets under the new master key **without changing their values**, so the CryptFs passphrase keeps deriving the same per-file keys and the encrypted files on disk are not affected. Changing the CryptFs passphrase value itself is a different operation: it re-derives the per-file keys and requires re-uploading the affected files (see [Data At Rest Encryption](dare.md)).

### Notes

- The KMS configuration is global.
- If you set a master key you will be unable to decrypt the data without this key and the SFTPGo users that need the data as plain text will be unable to login.
- You can start using the local provider and then switch to an external one but you can't switch between external providers and still be able to decrypt the data encrypted using the previous provider.
