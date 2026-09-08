---
description: "SFTPGo SSH/SFTP server: host keys, ciphers, KEX algorithms, public key authentication, and post-quantum hybrid key exchange (mlkem768x25519)."
---

# SSH/SFTP

SFTPGo provides a full-featured SFTP server built on the SSH protocol. Shell login and port forwarding are not supported — the SSH server is dedicated to file transfer operations.

## SFTP

The SFTP implementation supports [version 3](https://datatracker.ietf.org/doc/html/draft-ietf-secsh-filexfer-02){:target="_blank"} of the SFTP protocol, the same version used by OpenSSH. All standard file operations are supported: read, write, rename, delete, create directories, list directories, stat, and setstat.

The `statvfs@openssh.com` extension is also supported, allowing clients to query available disk space.

## Authentication

SFTPGo supports multiple authentication methods, which can be combined for multi-step authentication:

| Method | Description |
| -------- | ------------- |
| **Password** | Standard password authentication. Can be disabled per-user or globally. |
| **Public key** | SSH public key authentication. Multiple keys per user are supported, each one can be [restricted to source addresses](#restricting-a-key-to-source-addresses). |
| **Certificate** | SSH certificates signed by a trusted Certificate Authority. Supports principal validation, expiration, source-address restrictions, and revocation lists. See [configuration](config-file.md) for `trusted_user_ca_keys` and `revoked_user_certs_file`. |
| **Keyboard-interactive** | Challenge-response authentication. Supports external hooks for custom authentication flows (e.g., OTP, security questions). See [Keyboard Interactive Authentication](keyboard-interactive.md). |
| **OpenPubkey (OPKSSH)** | [OpenPubkey SSH](https://github.com/openpubkey/opkssh){:target="_blank"} integration via external binary. Mutually exclusive with certificate authentication. |

### Restricting a key to source addresses

A public key can be limited to the networks a client may use it from. Write the `from` option of the `authorized_keys` format before the key:

```
from="192.0.2.0/24,2001:db8::/32" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... build-server
```

The option lists one or more networks in CIDR notation, separated by commas. A single address takes its full prefix length, `192.0.2.10/32` or `2001:db8::10/128`. The other forms of the OpenSSH `from` option, hostnames and `!` negations, and the other `authorized_keys` options are refused when the user is saved, and the error reports the accepted syntax.

From an address outside the listed networks the key does not authenticate; the other keys of the user, if any, are tried as usual. A network covers its own address family: `0.0.0.0/0` covers IPv4 and `::/0` covers IPv6.

The address compared is the client address of the connection. Behind a proxy, enable the [PROXY protocol](config-file.md#proxy_protocol-modes) so that the address is the client's and not the proxy's. The [allowed and denied IP addresses](access-control.md#ip-and-protocol-restrictions) of the user keep applying.

SSH certificates are restricted with their own `source-address` option, set when the certificate is issued.

:warning: The restriction belongs to the key, not to the user: a user allowed to manage its public keys from the Web Client or the REST API can add a key without it. To make it binding, disable the change of the public keys with the matching Web client/REST API option, see [Web Client and REST API](users.md#web-client-and-rest-api).

### Multi-step authentication

Authentication methods can be chained — for example, requiring both a public key **and** a password. When a user is configured for multi-step authentication, the SSH server returns a "partial success" after the first method succeeds, prompting the client for the next step.

Supported combinations (the first listed factor is the one the client must offer first):

- `publickey+password` — public key, then password
- `publickey+keyboard-interactive` — public key, then keyboard-interactive
- `password+publickey` — password, then public key.
- `keyboard-interactive+publickey` — keyboard-interactive, then public key.

Each combination is a separate entry in the `denied_login_methods` filter. To allow both orderings, leave both entries out of the denylist; clients can then choose either ordering. To force only one direction, deny the unwanted variant explicitly.

:information_source: When a single-step method is also allowed (for example, the user has neither `publickey` nor `publickey+password` in the denylist), the single-step method is sufficient on its own — the second factor is not required. To force multi-step, deny every single-step method (`publickey`, `password`, `keyboard-interactive`) and leave only the desired multi-step combinations enabled.

## Cryptographic algorithms

SFTPGo supports modern cryptographic algorithms with secure defaults. All algorithm lists are configurable via the [configuration file](config-file.md).

### Key exchange (KEX)

Default algorithms (in preference order):

1. `mlkem768x25519-sha256` (post-quantum hybrid)
2. `curve25519-sha256`
3. `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`
4. `diffie-hellman-group14-sha256`
5. `diffie-hellman-group-exchange-sha256`

SHA-512 based KEX algorithms are available but disabled by default because they are slow. Legacy algorithms (`diffie-hellman-group1-sha1`, `diffie-hellman-group14-sha1`) are available but disabled by default for security reasons.

### Ciphers

Default: `aes128-gcm`, `aes256-gcm`, `chacha20-poly1305`, `aes128-ctr`, `aes192-ctr`, `aes256-ctr`.

:warning: Additional ciphers (`aes*-cbc`, `3des-cbc`, `arcfour*`) are available but disabled by default — they are insecure and an active attacker can recover plaintext.

### MACs

Default: `hmac-sha2-256-etm@openssh.com`, `hmac-sha2-256`.

### Host keys

If no host keys are configured, SFTPGo automatically generates `id_rsa`, `id_ecdsa`, and `id_ed25519` keys in the configuration directory. Host certificates are also supported — the certificate's public key must match a configured private host key.

## SSH commands

SFTPGo supports a limited set of SSH commands for specific use cases. All commands are **emulated internally in Go** — SFTPGo never spawns the corresponding system binaries (`scp`, `md5sum`, `uname`, ...) and does not require them to be installed on the host. Operations are executed within the user's security context (permissions, quotas, and home directory restrictions apply) and work consistently across all storage backends, including remote and encrypted ones.

| Command | Default | Description |
| --------- | --------- | ------------- |
| `scp` | Enabled | Server-side SCP protocol implementation. Supports recursive transfers and remote-to-remote copy via `-3`. Wildcard expansion is not supported. Works with all storage backends. |
| `md5sum` | Enabled | Compute MD5 hash of a file. |
| `sha1sum` | Enabled | Compute SHA-1 hash of a file. |
| `sha256sum` | Enabled | Compute SHA-256 hash of a file. |
| `sha384sum` | — | Compute SHA-384 hash of a file. |
| `sha512sum` | — | Compute SHA-512 hash of a file. |
| `cd` | Enabled | Change directory (currently a no-op, for client compatibility). |
| `pwd` | Enabled | Print working directory (always returns `/`, for client compatibility). |
| `sftpgo-copy` | — | Server-side copy of files and directories. Usage: `sftpgo-copy <source> <destination>`. |
| `sftpgo-remove` | — | Server-side recursive removal of files and directories. Usage: `sftpgo-remove <path>`. Does not support removing across virtual folder boundaries. |
| `uname` | — | Synthetic response for client probes that issue `uname` over SSH (e.g. some backup integrations). SFTPGo does not execute the system `uname` binary; it returns a fixed `uname -a`-style string. The default base is `Linux sftpgo 1.0.0 #1 SMP SFTPGo x86_64 GNU/Linux` (no real host data exposed) and can be overridden via [`SFTPGO_HOOK__SSHD_UNAME_OUTPUT`](env-vars.md). Standard flags (`-a`, `-s`, `-n`, `-r`, `-v`, `-m`, `-p`, `-i`, `-o` and their long forms) extract the matching field from the base string. |

:information_source: OpenSSH 9.0 and later transfer over SFTP when `scp` is invoked; the server-side SCP implementation described here serves the clients that request the legacy protocol, with `scp -O` on those releases. Both paths are supported. The SFTP one resolves the destination first, so it requires the `list` [permission](access-control.md#per-directory-permissions) on the target directory.

Commands not in the default list must be explicitly enabled in the [configuration](config-file.md). Set `enabled_ssh_commands` to `*` to enable all supported commands.

:warning: Hash commands read the entire file to compute the digest. For remote storage backends, this means downloading the file; for encrypted storage, this means decrypting it. The read requires the `download` [permission](access-control.md#per-directory-permissions) on the file's parent directory.

## Security features

- **Algorithm enforcement** — Users can require specific algorithms via the `enforce_secure_algorithms` filter. Connections using weaker algorithms are rejected even if the algorithms are enabled server-wide.
- **Login banner** — Display a custom message before authentication by configuring `login_banner_file`.
- **Max authentication attempts** — Configurable limit on failed authentication attempts per connection.
- **Defender integration** — Failed logins, algorithm mismatches, and connection limit violations generate defender events for automatic banning. See [Defender](defender.md).
