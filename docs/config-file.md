---
description: "Reference for the open-source SFTPGo configuration file: all settings for SFTP, FTP/S, WebDAV, HTTP, storage backends, rate limiting, and more."
---

# Configuration file

The default configuration file is [sftpgo.json](https://github.com/drakkan/sftpgo/blob/main/sftpgo.json){:target="_blank"}. It is divided into the following sections.

## Common

Configuration parameters for the `common` section.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `idle_timeout` | integer | `15` | Time in minutes after which an idle client will be disconnected. `0` means disabled. |
| `upload_mode` | integer | `0` | Upload behavior flags. Can be combined (see below). Ignored for the SFTP backend if buffering is enabled. |
| `setstat_mode` | integer | `0` | `0`: normal mode (permission/owner/time change requests are executed). `1`: ignore mode (requests are silently ignored). `2`: ignore if not supported (permission/owner changes are silently ignored for cloud filesystems, executed for local/SFTP). |
| `rename_mode` | integer | `0` | `0`: renaming non-empty directories is not allowed for cloud storage providers (S3, GCS, Azure Blob). `1`: enable recursive renames for these providers. Recursive renames may be slow and are not atomic; partial renames and incorrect quota updates are possible on error. |
| `resume_max_size` | integer | `0` | Maximum file size in bytes for which upload resume is allowed on storage backends with immutable objects (S3, GCS, Azure Blob). SFTPGo must rewrite the entire file on resume. `0` means resume is disabled. |
| `temp_path` | string | empty | Path for temporary files (atomic uploads, file pipes). Must exist, be writable, and reside on the same filesystem as user home directories (otherwise atomic renames become copies). Temporary files are not namespaced. Leave empty for the default. |
| `proxy_protocol` | integer | `0` | [HAProxy PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt){:target="_blank"} support. Supported for SSH/SFTP and FTP/S. See modes below. |
| `proxy_allowed` | list of strings | empty | IP addresses and ranges allowed to send the proxy header. If `proxy_protocol` is `1`, connections from unlisted IPs are accepted but the header is ignored. If `proxy_protocol` is `2`, connections from unlisted IPs are rejected. |
| `proxy_skipped` | list of strings | empty | IP addresses and ranges for which the proxy header is not read. |
| `startup_hook` | string | empty | Absolute path to a program or an HTTP URL to invoke (via GET) when SFTPGo starts. Services may not yet be available when this hook runs. |
| `post_connect_hook` | string | empty | Absolute path to a command or HTTP URL to notify. See [Post-connect hook](post-connect-hook.md) for details. |
| `post_disconnect_hook` | string | empty | Absolute path to a command or HTTP URL to notify. See [Post-disconnect hook](post-disconnect-hook.md) for details. |
| `max_total_connections` | integer | `0` | Maximum number of concurrent client connections. `0` means unlimited. |
| `max_per_host_connections` | integer | `20` | Maximum concurrent client connections from the same IP. If the defender is enabled, exceeding this limit generates `score_limit_exceeded` events, and repeat offenders can be automatically banned. `0` means unlimited. |
| `allowlist_status` | integer | `0` | Set to `1` to enable the allow list. When enabled, only listed IPs/networks can access configured services; all other connections are dropped before authentication. Populate the list (via WebAdmin or REST API) before enabling. |
| `allow_self_connections` | integer | `0` | Allow users on this instance to use other users/virtual folders on the same instance as a storage backend. Set to `1` to enable. Only enable if you understand the implications. |
| `umask` | string | empty | File mode creation mask (e.g., `002`). Leave blank to use the system default. Supported on *NIX platforms only. |
| `server_version` | string | empty | Customize the advertised software version. `short` hides the version number; any other value falls back to the default. |
| `tz` | string | empty | Time zone for the EventManager scheduler and time-based access restrictions. Set to `local` for server local time; otherwise UTC is used. |

:information_source: The per-host and total connection caps are checked and applied without atomic coordination. Under bursts of near-simultaneous connections, the active count can briefly exceed the configured limit by a small number of connections before the rejection logic catches up. The window is narrow (no I/O between check and increment), so this is a hard cap only on average — not a strict hard cap on instantaneous peaks. If you need a strict atomic cap (e.g. to protect a downstream resource sized exactly to the limit), set the value below your target peak to leave headroom.

#### `upload_mode` flags

| Flag | Description |
| ------ | ------------- |
| `0` | **Standard**: files are uploaded directly to the requested path. |
| `1` | **Atomic**: files are uploaded to a temporary path and renamed on success. On error, the temporary file is deleted. Avoids serving partial files. |
| `2` | **Atomic with resume**: same as atomic, but on error the temporary file is renamed to the requested path (not deleted), allowing clients to resume. If both `1` and `2` are provided, `2` takes precedence. |
| `4` | Store partial files for the S3 backend even if a client-side upload error is detected. |
| `8` | Store partial files for the Google Cloud Storage backend even if a client-side upload error is detected. |
| `16` | Store partial files for the Azure Blob backend even if a client-side upload error is detected. |

#### `proxy_protocol` modes

| Mode | Description |
| ------ | ------------- |
| `0` | Disabled. |
| `1` | Enabled. The proxy header is always read. If the upstream IP is not in `proxy_allowed`, the header is ignored (but the connection is accepted). Use `proxy_skipped` to allow specific IPs/networks to connect without sending or having their proxy header read. |
| `2` | Required. The proxy header is always read. If the upstream IP is not in `proxy_allowed` and a proxy header is found, the connection is rejected. Use `proxy_skipped` to allow specific IPs/networks to connect without sending or having their proxy header read. |

#### `actions`

External action hooks for file operations. See [Custom Actions](custom-actions.md) for details.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `execute_on` | list of strings | empty | Trigger conditions. Valid values: `pre-download`, `download`, `first-download`, `pre-upload`, `upload`, `first-upload`, `pre-delete`, `delete`, `rename`, `mkdir`, `rmdir`, `ssh_cmd`, `copy`. Leave empty to disable actions. |
| `execute_sync` | list of strings | empty | Actions from `execute_on` to execute synchronously. The `pre-*` actions are always synchronous. Synchronous execution means SFTPGo will not return a result to the client until the hook completes. |
| `hook` | string | empty | Absolute path to a command or HTTP URL to invoke. |

#### `metadata`

Configuration for Cloud Storage backend metadata.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `read` | integer | `0` | Set to `1` to read metadata before downloading files from Cloud Storage backends and make it available in notification events. |

#### `defender`

Defender configuration for automatic banning of misbehaving clients. See [Defender](defender.md) for details.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `enabled` | boolean | `false` | Enable the defender. |
| `driver` | string | `memory` | `memory` or `provider`. The `provider` driver stores events in the data provider (supported for MySQL, PostgreSQL, and CockroachDB), allowing shared state across multiple SFTPGo instances. For a single instance, `memory` is much faster. |
| `ban_time` | integer | `30` | Ban duration in minutes. |
| `ban_time_increment` | integer | `50` | Ban time increment as a percentage if a banned host tries to connect again. |
| `threshold` | integer | `15` | Score threshold for banning a client. |
| `score_invalid` | integer | `2` | Score for invalid login attempts (e.g., non-existent user accounts). |
| `score_valid` | integer | `1` | Score for valid login attempts (e.g., existing user accounts with wrong password). |
| `score_limit_exceeded` | integer | `3` | Score for hosts that exceeded rate limits or per-host connection limits. |
| `score_no_auth` | integer | `0` | Score for clients disconnected without any authentication attempt. |
| `observation_time` | integer | `30` | Time window in minutes for tracking client errors. A host is banned if its score exceeds the threshold within this period. |
| `entries_soft_limit` | integer | `100` | Ignored for the `provider` driver. |
| `entries_hard_limit` | integer | `150` | For the `memory` driver, the number of banned IPs and host scores kept in memory varies between the soft and hard limits. For the `provider` driver, this limits the number of entries returned when listing all defender hosts. |

#### `defender.login_delay`

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `success` | integer | `0` | Milliseconds to pause before allowing a successful login. `0` means disabled. |
| `password_failed` | integer | `1000` | Milliseconds to pause before reporting a failed password/interactive login. `0` means disabled. |

#### `rate_limiters`

List of rate limiter configurations. See [Rate Limiting](rate-limiting.md) for details.

Each entry in the list has the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `average` | integer | `0` | Maximum allowed rate. `0` means disabled. |
| `period` | integer | `1000` | Period in milliseconds. The effective rate is `average / period`. |
| `burst` | integer | `1` | Maximum number of requests allowed in an arbitrarily small time window. |
| `type` | integer | `2` | `1`: global rate limiter (independent of source host). `2`: per-IP rate limiter. |
| `protocols` | list of strings | all | Protocols to apply the limiter to. Available: `SSH`, `FTP`, `DAV`, `HTTP`. |
| `generate_defender_events` | boolean | `false` | If `true` and the defender is enabled and this is a per-IP limiter, a defender event is generated each time the limit is exceeded. |
| `entries_soft_limit` | integer | `100` | Soft limit for per-IP rate limiters kept in memory. |
| `entries_hard_limit` | integer | `150` | Hard limit. The number of per-IP rate limiters in memory varies between the soft and hard limits. |

#### `event_manager`

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `enabled_commands` | list of strings | empty | Absolute paths to system commands that can be executed through the EventManager. :warning: Allowing system commands could pose a security risk. |

## ACME

Automatic Certificate Management Environment (ACME) protocol configuration. To obtain certificates for the first time, configure the ACME protocol and run the `sftpgo acme run` command or use the WebAdmin UI. The SFTPGo service handles automatic renewal of certificates for the configured domains.

Configuration parameters for the `acme` section:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `domains` | list of strings | empty | Domains for which to obtain certificates. To have a single certificate valid for multiple domains, separate names with commas or spaces (e.g., `example.com,www.example.com`). An empty list disables the ACME protocol. |
| `email` | string | empty | Email used for registration and recovery contact. |
| `key_type` | string | `4096` | Key type for private keys. Supported values: `2048` (RSA 2048), `3072` (RSA 3072), `4096` (RSA 4096), `8192` (RSA 8192), `P256` (EC 256), `P384` (EC 384). |
| `certs_path` | string | `certs` | Directory for storing certificates and related data. Can be absolute or relative to the configuration directory. |
| `ca_endpoint` | string | `https://acme-v02.api.letsencrypt.org/directory` | ACME Certificate Authority endpoint URL. |
| `renew_days` | integer | `30` | Number of days before expiration at which to renew the certificate. |

#### `http01_challenge`

Configuration for the `HTTP-01` challenge type.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `port` | integer | `80` | Port for the HTTP-01 challenge. This challenge is expected to run on port 80. If you use a different port, you must proxy the path `/.well-known/acme-challenge` from port 80 to the configured port. |
| `proxy_header` | string | empty | HTTP header to validate against when solving HTTP-based challenges behind a reverse proxy. Empty means `Host`. |
| `webroot` | string | empty | Absolute path to a webroot folder for HTTP-based challenges. When set, the built-in server is disabled (`port` is ignored) and the directory must be publicly served on port 80 with access to `.well-known/acme-challenge`. If both `webroot` is empty and `port` is `0`, the HTTP-01 challenge is disabled. |

#### `tls_alpn01_challenge`

Configuration for the `TLS-ALPN-01` challenge type.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `port` | integer | `0` | Port for the TLS-ALPN-01 challenge. This challenge is expected to run on port 443. `0` disables TLS-ALPN-01. |

## SSH/SFTP server

Configuration parameters for the `sftpd` section:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `max_auth_tries` | integer | `0` | Maximum number of authentication attempts per connection. If set to a negative number, the number of attempts is unlimited. If set to zero, the number of attempts is limited to 6. |
| `host_keys` | list of strings | empty | Daemon's private host key paths (absolute or relative to the config directory). If empty, the daemon searches for or generates `id_rsa`, `id_ecdsa`, and `id_ed25519` keys inside the configuration directory. If you configure absolute paths to files named `id_rsa`, `id_ecdsa`, and/or `id_ed25519`, SFTPGo will try to generate these keys using default settings. |
| `host_certificates` | list of strings | empty | Public host certificate paths (absolute or relative to the config directory). A certificate's public key must match a private host key; otherwise it is silently ignored. |
| `host_key_algorithms` | list of strings | `rsa-sha2-256`, `rsa-sha2-512`, `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `ssh-ed25519` | Public key algorithms the server accepts for host key authentication. Supported values: `rsa-sha2-512-cert-v01@openssh.com`, `rsa-sha2-256-cert-v01@openssh.com`, `ssh-rsa-cert-v01@openssh.com`, `ssh-dss-cert-v01@openssh.com`, `ecdsa-sha2-nistp256-cert-v01@openssh.com`, `ecdsa-sha2-nistp384-cert-v01@openssh.com`, `ecdsa-sha2-nistp521-cert-v01@openssh.com`, `ssh-ed25519-cert-v01@openssh.com`, `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `rsa-sha2-512`, `rsa-sha2-256`, `ssh-rsa`, `ssh-dss`, `ssh-ed25519`. Certificate algorithms are listed for backward compatibility only and are not used. |
| `kex_algorithms` | list of strings | `mlkem768x25519-sha256`, `curve25519-sha256`, `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`, `diffie-hellman-group14-sha256`, `diffie-hellman-group-exchange-sha256` | KEX (Key Exchange) algorithms in preference order. Leave empty for defaults. Supported values: `mlkem768x25519-sha256`, `curve25519-sha256` (also enables alias `curve25519-sha256@libssh.org`), `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`, `diffie-hellman-group14-sha256`, `diffie-hellman-group16-sha512`, `diffie-hellman-group14-sha1`, `diffie-hellman-group1-sha1`, `diffie-hellman-group-exchange-sha256`, `diffie-hellman-group-exchange-sha1`. SHA-512 based KEXs are disabled by default because they are slow. |
| `ciphers` | list of strings | `aes128-gcm@openssh.com`, `aes256-gcm@openssh.com`, `chacha20-poly1305@openssh.com`, `aes128-ctr`, `aes192-ctr`, `aes256-ctr` | Allowed ciphers in preference order. Leave empty for defaults. Supported values: `aes128-gcm@openssh.com`, `aes256-gcm@openssh.com`, `chacha20-poly1305@openssh.com`, `aes128-ctr`, `aes192-ctr`, `aes256-ctr`, `aes128-cbc`, `aes192-cbc`, `aes256-cbc`, `3des-cbc`, `arcfour256`, `arcfour128`, `arcfour`. :warning: Ciphers disabled by default are insecure; an active attacker can recover plaintext if you enable them. |
| `macs` | list of strings | `hmac-sha2-256-etm@openssh.com`, `hmac-sha2-256` | MAC (Message Authentication Code) algorithms in preference order. Leave empty for defaults. Supported values: `hmac-sha2-256-etm@openssh.com`, `hmac-sha2-256`, `hmac-sha2-512-etm@openssh.com`, `hmac-sha2-512`, `hmac-sha1`, `hmac-sha1-96`. |
| `public_key_algorithms` | list of strings | `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `rsa-sha2-512`, `rsa-sha2-256`, `ssh-ed25519`, `sk-ssh-ed25519@openssh.com`, `sk-ecdsa-sha2-nistp256@openssh.com` | Public key algorithms the server accepts for client authentication. Supported values: `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `rsa-sha2-512`, `rsa-sha2-256`, `ssh-rsa`, `ssh-dss`, `ssh-ed25519`, `sk-ssh-ed25519@openssh.com`, `sk-ecdsa-sha2-nistp256@openssh.com`. |
| `trusted_user_ca_keys` | list of strings | empty | Paths to public keys of certificate authorities trusted to sign user certificates for authentication. Paths can be absolute or relative to the configuration directory. |
| `revoked_user_certs_file` | string | empty | Path to a JSON file containing revoked user certificate fingerprints (absolute or relative to the config directory). Example: `["SHA256:bsBRHC/xgiqBJdSuvSTNpJNLTISP/G356jNMCRYC5Es","SHA256:119+8cL/HH+NLMawRsJx6CzPF1I3xC+jpM60bQHXGE8"]`. The list can be reloaded on demand via `SIGHUP` on Unix or a `paramchange` request on Windows. |
| `opkssh_path` | string | empty | Absolute path to the `opkssh` binary for [OpenPubkey SSH](https://github.com/openpubkey/opkssh){:target="_blank"} integration. Ensure the SFTPGo service user has permissions to access the binary and its configuration files. Mutually exclusive with `trusted_user_ca_keys`. |
| `opkssh_checksum` | string | empty | Expected SHA256 checksum of the `opkssh` binary, verified at application startup. |
| `login_banner_file` | string | empty | Path to a login banner file (absolute or relative to the config directory). Contents are sent to the remote user before authentication. Leave empty to disable. |
| `enabled_ssh_commands` | list of strings | `md5sum`, `sha1sum`, `sha256sum`, `cd`, `pwd`, `scp` | List of enabled SSH commands. `*` enables all supported commands. [More information](ssh.md#ssh-commands). |
| `keyboard_interactive_authentication` | boolean | `true` | Whether keyboard interactive authentication is allowed. If no keyboard interactive hook or auth plugin is defined, the default behavior prompts for the user password followed by the one-time authentication code, if defined. |
| `keyboard_interactive_auth_hook` | string | empty | Absolute path to an external program or an HTTP URL to invoke for keyboard interactive authentication. See [Keyboard Interactive Authentication](keyboard-interactive.md) for details. |
| `password_authentication` | boolean | `true` | Set to `false` to disable password authentication. This also disables multi-step authentication (public key + password). Useful for public-key-only configurations when managing older clients that skip public key auth if password login is advertised. |

#### `bindings`

List of listener bindings for the SSH/SFTP server. Each entry supports the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `port` | integer | `2022` | Port for serving SFTP requests. `0` disables the binding. |
| `address` | string | empty | Listen address. Leave blank to listen on all available network interfaces. |
| `apply_proxy_config` | boolean | `true` | If enabled, the common proxy configuration is applied to this binding. |

## FTP server

Configuration parameters for the `ftpd` section:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `banner_file` | string | empty | Path to a banner file (absolute or relative to the config directory). Contents are displayed when a client connects. Leave empty to disable. |
| `active_transfers_port_non_20` | boolean | `true` | Do not require port 20 for active data transfers. Enabling this allows running SFTPGo with fewer privileges. |
| `passive_port_range` | struct | `start: 50000`, `end: 50100` | Port range for passive data connections (keys: `start`, `end`). Random if not specified. |
| `disable_active_mode` | boolean | `false` | Set to `true` to disable active FTP mode. |
| `enable_site` | boolean | `false` | Set to `true` to enable the FTP `SITE` command. Supports `chmod` and `symlink` when enabled. |
| `hash_support` | integer | `0` | Set to `1` to enable FTP hash commands: `HASH`, `XCRC`, `MD5/XMD5`, `XSHA/XSHA1`, `XSHA256`, `XSHA512`. :warning: Calculating a hash requires reading the entire file. For remote backends this means downloading the file; for encrypted backends this means decrypting it. |
| `combine_support` | integer | `0` | Set to `1` to enable the non-standard `COMB` FTP command. Only supported for local filesystem; for cloud backends it offers no advantage as it downloads partial files and re-uploads the combined result. Cloud backends natively support multipart uploads. |
| `certificate_file` | string | empty | TLS certificate for FTPS (absolute or relative to the config directory). |
| `certificate_key_file` | string | empty | Private key matching the above certificate (absolute or relative to the config directory). Both a certificate and key are required to enable explicit and implicit TLS. Files can be reloaded via `SIGHUP` on Unix or `paramchange` on Windows. Certificates are also polled for changes every 8 hours. |
| `ca_certificates` | list of strings | empty | Root certificate authorities used to verify client certificates. |
| `ca_revocation_lists` | list of strings | empty | Revocation lists (one per root CA) used to check if a client certificate has been revoked. Can be reloaded via `SIGHUP` on Unix or `paramchange` on Windows. |

#### `bindings`

List of listener bindings for the FTP server. Each entry supports the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `port` | integer | `0` | Port for serving FTP requests. `0` disables the binding. |
| `address` | string | empty | Listen address. Leave blank to listen on all available network interfaces. |
| `apply_proxy_config` | boolean | `true` | If enabled, the common proxy configuration is applied. :warning: The proxy header is expected on both control and data connections. |
| `tls_mode` | integer | `0` | TLS mode. `0`: accept both cleartext and encrypted sessions. `1`: TLS required for both control and data connections. `2`: implicit TLS. Ensure a proper TLS configuration is in place when setting a value other than `0`. |
| `certificate_file` | string | empty | Binding-specific TLS certificate (absolute or relative to the config directory). |
| `certificate_key_file` | string | empty | Binding-specific private key matching the above certificate (absolute or relative to the config directory). If not set, the global certificate and key are used, if any. |
| `min_tls_version` | integer | `12` | Minimum TLS version. `12`: TLS 1.2+, `13`: TLS 1.3, `10`: TLS 1.0, `11`: TLS 1.1. |
| `force_passive_ip` | string | empty | External IPv4 address for passive connections. Leave empty to auto-detect. Must be a valid IPv4 address if set. |
| `passive_host` | string | empty | Hostname for passive connections. Resolved on each passive connection request, which may introduce noticeable latency depending on DNS configuration. Enable only if you have a dynamic IP address. |
| `client_auth_type` | integer | `0` | Client certificate requirements. `0`: no client certificate required. `1`: require and verify a client certificate. `2`: request a client certificate and verify it if provided; the client may omit the certificate. At least one CA must be defined to verify client certificates; otherwise this setting is ignored. |
| `tls_cipher_suites` | list of strings | empty | Cipher suites for TLS 1.2 and below. If empty, a default list of secure ciphers is used with hardware-performance-based preference order. TLS 1.3 cipher suites are not configurable. See supported [cipher suite names](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Invalid names are silently ignored. Order matters: ciphers listed first are preferred. |
| `passive_connections_security` | integer | `0` | Security checks for passive data connections. `0`: require matching peer IP addresses for control and data connections. `1`: disable checks. :warning: When running FTP behind a proxy, you must enable the proxy protocol for both control and data connections. |
| `active_connections_security` | integer | `0` | Security checks for active data connections (same values as `passive_connections_security`). :warning: Disabling security checks makes the FTP service vulnerable to bounce attacks on active data connections. Change the default only on trusted/internal networks. |
| `debug` | boolean | `false` | If enabled, every FTP command is logged. :warning: This generates a large volume of logs. Enable only for investigating client compatibility issues; do not leave enabled in production. |

#### `passive_ip_overrides`

Nested within each binding. Allows returning a different passive IP based on the client IP address. Each entry has the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `networks` | list of strings | — | Networks in CIDR notation (e.g., `192.168.1.0/24`). |
| `ip` | string | empty | Passive IP to return if the client IP belongs to the defined networks. Empty means auto-detect. |

## WebDAV Server

Configuration parameters for the `webdavd` section:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `certificate_file` | string | empty | TLS certificate for WebDAV over HTTPS (absolute or relative to the config directory). |
| `certificate_key_file` | string | empty | Private key matching the above certificate (absolute or relative to the config directory). Both are required to enable HTTPS. Files can be reloaded via `SIGHUP` on Unix or `paramchange` on Windows. |
| `ca_certificates` | list of strings | empty | Root certificate authorities used to verify client certificates. |
| `ca_revocation_lists` | list of strings | empty | Revocation lists (one per root CA) used to check if a client certificate has been revoked. Can be reloaded via `SIGHUP` on Unix or `paramchange` on Windows. Certificates are also polled for changes every 8 hours. |

#### `bindings`

List of listener bindings for the WebDAV server. Each entry supports the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `port` | integer | `0` | Port for serving WebDAV requests. `0` disables the binding. |
| `address` | string | empty | Listen address. Leave blank to listen on all available network interfaces. |
| `enable_https` | boolean | `false` | Set to `true` and provide a certificate and key file to enable HTTPS for this binding. |
| `certificate_file` | string | empty | Binding-specific TLS certificate (absolute or relative to the config directory). |
| `certificate_key_file` | string | empty | Binding-specific private key matching the above certificate (absolute or relative to the config directory). If not set, the global certificate and key are used, if any. |
| `min_tls_version` | integer | `12` | Minimum TLS version. `12`: TLS 1.2+, `13`: TLS 1.3, `10`: TLS 1.0, `11`: TLS 1.1. |
| `client_auth_type` | integer | `0` | Client certificate requirements. `0`: no client certificate required. `1`: require and verify a client certificate. `2`: request a client certificate and verify it if provided; the client may omit the certificate. At least one CA must be defined to verify client certificates; otherwise this setting is ignored. |
| `tls_cipher_suites` | list of strings | empty | Cipher suites for TLS 1.2 and below. If empty, a default list of secure ciphers is used with hardware-performance-based preference order. TLS 1.3 cipher suites are not configurable. See supported [cipher suite names](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Invalid names are silently ignored. Order matters: ciphers listed first are preferred. |
| `tls_protocols` | list of strings | `http/1.1`, `h2` | HTTPS protocols in preference order. Supported values: `http/1.1`, `h2`. |
| `prefix` | string | empty | Prefix for WebDAV resources. If empty, resources are available at `/`. If defined, it must be an absolute URI (e.g., `/dav`). |
| `proxy_mode` | integer | `0` | Set to `1` to use the proxy protocol configuration from the `common` section instead of the proxy header configuration. |
| `proxy_allowed` | list of strings | empty | IP addresses and ranges allowed to set the client IP proxy header (e.g., `X-Forwarded-For`). Proxy headers from connections not in this list are silently ignored. |
| `client_ip_proxy_header` | string | empty | Client IP proxy header to trust (e.g., `X-Forwarded-For`, `X-Real-IP`). |
| `client_ip_header_depth` | integer | `0` | Position to trust in multi-value client IP headers (e.g., `X-Forwarded-For`), counting from the right. For `10.0.0.1,11.0.0.1,12.0.0.1,13.0.0.1`: depth `0` uses `13.0.0.1`, depth `1` uses `12.0.0.1`. Set to `-1` to trust the leftmost IP address. :warning: Using `-1` may have security implications and should only be used if your proxy has appropriate controls to prevent spoofed IP headers. |
| `disable_www_auth_header` | boolean | `false` | Set to `true` to omit the `WWW-Authenticate` header after an authentication failure; only the `401` status code will be sent. |

#### `cors`

CORS (Cross-Origin Resource Sharing) configuration. SFTPGo uses [Go CORS handler](https://github.com/rs/cors){:target="_blank"}; refer to upstream documentation for detailed field semantics and default values.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `enabled` | boolean | `false` | Set to `true` to enable CORS. |
| `allowed_origins` | list of strings | empty | Origins allowed to make cross-origin requests. |
| `allowed_methods` | list of strings | empty | HTTP methods allowed for cross-origin requests. |
| `allowed_headers` | list of strings | empty | Headers allowed in cross-origin requests. |
| `exposed_headers` | list of strings | empty | Headers exposed to the browser in cross-origin responses. |
| `allow_credentials` | boolean | `false` | Whether credentials (cookies, authorization headers) are allowed. |
| `max_age` | integer | `0` | Maximum time (in seconds) the preflight response can be cached. |
| `options_passthrough` | boolean | `false` | Whether OPTIONS requests are passed through to the application. |
| `options_success_status` | integer | `0` | HTTP status code for successful OPTIONS requests. |
| `allow_private_network` | boolean | `false` | Whether to allow private network access in CORS requests. |

#### `cache`

Cache configuration for the WebDAV server.

##### `cache.users`

Cache configuration for authenticated users.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `expiration_time` | integer | `0` | Expiration time in minutes for cached users. `0` means unlimited. |
| `max_size` | integer | `50` | Maximum number of users to cache. `0` means unlimited. |

##### `cache.mime_types`

Cache configuration for MIME types.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `enabled` | boolean | `true` | Set to `true` to enable MIME type caching. |
| `max_size` | integer | `1000` | Maximum number of MIME types to cache. `0` disables caching. |

##### `cache.mime_types.custom_mappings`

Additional MIME type mappings. This provides a platform-independent way to add a small number of extra mappings. For large lists, use your operating system's native method. Each entry has the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `ext` | string | — | File extension including the dot (e.g., `.json`). |
| `mime` | string | — | MIME type (e.g., `application/json`). |

## Data provider

Supported configuration parameters for the `data_provider` section:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `driver` | string | `sqlite` | Supported drivers are `sqlite`, `mysql`, `postgresql`, `cockroachdb`, `bolt`, `memory`. |
| `name` | string | `sftpgo.db` | Database name. For `sqlite`, this can be a database name relative to the config dir or an absolute path to the SQLite database. For `memory`, this is the optional path (relative to the config dir or absolute) to a provider dump obtained via the `dumpdata` REST API. The dump is loaded at startup and can be reloaded on demand by sending a `SIGHUP` signal on Unix-based systems or a `paramchange` request to the running service on Windows. The `memory` provider does not modify the provided file, so quota usage and last login are not persisted. If you plan to use a SQLite database over a `cifs` network share (not recommended), you must use the `nobrl` mount option to avoid the `database is locked` error. Some users have reported that the `bolt` provider works fine over `cifs` shares. |
| `host` | string | empty | Database host. For `postgresql` and `cockroachdb` drivers, you can specify multiple hosts separated by commas. Leave empty for `sqlite`, `bolt`, and `memory`. |
| `port` | integer | `0` | Database port. Leave empty for `sqlite`, `bolt`, and `memory`. |
| `username` | string | empty | Database user. Leave empty for `sqlite`, `bolt`, and `memory`. |
| `password` | string | empty | Database password. Leave empty for `sqlite`, `bolt`, and `memory`. |
| `sslmode` | integer | `0` | Used for `mysql` and `postgresql` drivers. `0` disables TLS. `1` requires TLS. `2` sets TLS mode to `verify-ca` (postgresql) or `skip-verify` (mysql). `3` sets TLS mode to `verify-full` (postgresql) or `preferred` (mysql). `4` sets TLS mode to `prefer` (postgresql only). `5` sets TLS mode to `allow` (postgresql only). |
| `root_cert` | string | empty | Path to the root certificate authority used to verify that the server certificate was signed by a trusted CA. |
| `disable_sni` | boolean | `false` | Allows opting out of Server Name Indication (SNI) for TLS connections. |
| `target_session_attrs` | string | empty | PostgreSQL and CockroachDB specific. Determines whether the session must have certain properties to be acceptable. Typically used with multiple host names to select the first acceptable alternative. Supported values: `any`, `read-write`, `read-only`, `primary`, `standby`, `prefer-standby`. If empty, `any` is assumed. If you explicitly set `any`, connections are randomly distributed among the specified hosts. |
| `client_cert` | string | empty | Path to the client certificate for two-way TLS authentication. |
| `client_key` | string | empty | Path to the client key for two-way TLS authentication. |
| `connection_string` | string | empty | Custom database connection string. If not empty, this is used instead of building one from the previous parameters. Leave empty for `bolt` and `memory`. |
| `sql_tables_prefix` | string | empty | Prefix for SQL tables. |
| `track_quota` | integer | `2` | Preferred mode to track user quota: `0` disables quota tracking (quota scan REST API does nothing). `1` updates quota on every upload or delete, even for users without quota restrictions. `2` updates quota on every upload or delete, but only for users with quota restrictions and for virtual folders (quota scan REST API can still be used for other users/folders). |
| `delayed_quota_update` | integer | `0` | Number of seconds to accumulate quota updates. Accumulating updates can reduce data provider queries when uploads are frequent. A scheduled quota update is recommended regardless, since stored quota may become incorrect due to unexpected shutdowns, temporary provider failures, files copied outside of SFTPGo, etc. Use the [event manager](eventmanager.md) to schedule periodic quota updates. `0` means immediate quota update. |
| `pool_size` | integer | `0` | Maximum number of open connections for `mysql` and `postgresql` drivers. `0` means unlimited. |
| `users_base_dir` | string | empty | Default base directory for users. If no home directory is defined when adding a new user, and this value is a valid absolute path, the user home directory is automatically set to `users_base_dir/username`. |
| `external_auth_hook` | string | empty | Absolute path to an external program or an HTTP URL to invoke for user authentication. See [External Authentication](external-auth.md) for details. Leave empty to disable. |
| `external_auth_scope` | integer | `0` | `0` means all supported scopes (passwords, public keys, and keyboard interactive). `1` passwords only. `2` public keys only. `4` keyboard interactive only. `8` TLS certificate. Flags can be combined (e.g., `6` means public keys and keyboard interactive). |
| `pre_login_hook` | string | empty | Absolute path to an external program or an HTTP URL to invoke to modify user details just before login. See [Dynamic user modification](dynamic-user-mod.md) for details. Leave empty to disable. |
| `post_login_hook` | string | empty | Absolute path to an external program or an HTTP URL to invoke to notify on login. See [Post-login hook](post-login-hook.md) for details. Leave empty to disable. |
| `post_login_scope` | integer | `0` | Scope for the post-login hook. `0` notifies on both failed and successful logins. `1` failed logins only. `2` successful logins only. |
| `check_password_hook` | string | empty | Absolute path to an external program or an HTTP URL to invoke to check user-provided passwords. See [Check password hook](check-password-hook.md) for details. Leave empty to disable. |
| `check_password_scope` | integer | `0` | Scope for the check password hook. `0` means all protocols. `1` SSH. `2` FTP. `4` WebDAV. Flags can be combined (e.g., `6` means FTP and WebDAV). |
| `password_caching` | boolean | `true` | Enables in-memory password caching. Verifying argon2id and bcrypt passwords is computationally expensive; caching reduces this cost. |
| `update_mode` | integer | `0` | Defines how the database is initialized/updated. `0` means automatically. `1` means manually using the `initprovider` sub-command. |
| `create_default_admin` | boolean | `false` | Automatically create the first admin account using the environment variables `SFTPGO_DEFAULT_ADMIN_USERNAME` and `SFTPGO_DEFAULT_ADMIN_PASSWORD`. You can also create the first admin by loading initial data. Has no effect if an admin account already exists. |
| `naming_rules` | integer | `5` | Naming rules for usernames, folders, groups, roles, and object names. `0` no rules. `1` allows any UTF-8 character (names are used in URIs for REST API and WebAdmin; without this, only unreserved URI characters are allowed: `ALPHA / DIGIT / - . _ ~`). `2` converts names to lowercase before saving/matching (enables case-insensitive matching). `4` trims trailing and leading whitespace before saving/matching (required for WebAdmin to work properly). Flags can be combined (e.g., `3` means lowercase + any UTF-8). :warning: Enabling these options on existing installations may be backward incompatible; ensure all existing users respect the defined rules. Control characters, `/`, and `\` are never permitted. |
| `is_shared` | integer | `0` | Set to `1` if the data provider is shared across multiple SFTPGo instances. Only `MySQL`, `PostgreSQL`, and `CockroachDB` can be shared; this setting is ignored for other providers. When shared, active transfers are persisted in the database (enabling cross-instance quota checks), and password reset requests and OIDC tokens/states are also persisted. Scheduled event actions run on a single instance by default (overridable per action). The `shared_sessions` table stores temporary sessions; in performance-critical installations, consider database-specific optimizations (e.g., `UNLOGGED` tables for PostgreSQL). |
| `backups_path` | string | `backups` | Path to the backup directory. Can be absolute or relative to the config dir. Arbitrary paths are not allowed for security reasons. |

#### actions

Configuration for commands or HTTP notifications triggered by provider events. See [Custom Actions](custom-actions.md) for details.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `execute_on` | list of strings | empty | Valid values: `add`, `update`, `delete`. The `update` action is not fired for internal updates such as last login or quota fields. |
| `execute_for` | list of strings | empty | Provider objects that trigger the action. Valid values: `user`, `folder`, `group`, `admin`, `api_key`, `share`, `event_action`, `event_rule`, `role`, `ip_list_entry`, `configs`. |
| `hook` | string | empty | Absolute path to the command to execute or HTTP URL to notify. |

#### password_hashing

Configuration for password hash generation. SFTPGo can verify passwords in several formats and uses `bcrypt` by default to hash plain-text passwords before storing them.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `algo` | string | `bcrypt` | Algorithm for hashing passwords. Available: `argon2id`, `bcrypt`. For bcrypt, the `$2a$` prefix is used. |

**argon2_options** -- Options for the argon2id hashing algorithm. Higher `memory` and `iterations` values increase security but also computational cost. On multi-core machines, increasing `parallelism` can reduce runtime without reducing cost.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `memory` | unsigned integer | `65536` | Amount of memory used by the algorithm (in kibibytes). |
| `iterations` | unsigned integer | `1` | Number of iterations over the memory. |
| `parallelism` | unsigned 8-bit integer | `2` | Number of threads (or lanes) used by the algorithm. |

**bcrypt_options** -- Options for the bcrypt hashing algorithm.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `cost` | integer | `10` | Cost parameter (valid range: 4--31). |

#### password_validation

Password validation rules. Any value set to `0` disables the corresponding check.

The `admins` and `users` sub-sections behave differently:

- **`admins`** — minimum password entropy enforced for admin accounts. Admin accounts have no per-admin password fields, so this value is always enforced.
- **`users`** — fallback minimum entropy for protocol-user passwords, used when neither the user nor the primary group defines a `password_strength`.

**admins** -- Minimum password rules enforced for SFTPGo admin accounts. There is no way to override this value per admin.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `min_entropy` | float | `0` | Minimum password entropy. [More details](https://github.com/wagslane/go-password-validator#what-entropy-value-should-i-use){:target="_blank"}. `0` means disabled (any password accepted). |

**users** -- Default password rules for SFTPGo protocol users.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `min_entropy` | float | `0` | Default minimum password entropy when the user and primary group do not set `password_strength`. |

#### node

Node-specific configuration for inter-node communication. If your provider is shared across multiple nodes, nodes can exchange information to present a uniform view for node-specific data (e.g., active connections from all nodes). Nodes connect to each other using the REST API.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `host` | string | empty | IP address or hostname that other nodes can use to connect to this node via REST API. Empty means inter-node communication is disabled. |
| `port` | integer | `0` | Port that other nodes can use to connect to this node via REST API. |
| `proto` | string | `http` | Supported values: `http`, `https`. For `https`, the HTTP client configuration is used (e.g., mutual TLS authentication). |

---

## HTTP server

Supported configuration parameters for the `httpd` section (REST API, WebAdmin, WebClient):

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `templates_path` | string | `templates` | Path to the HTML web templates. Can be absolute or relative to the config dir. |
| `static_files_path` | string | `static` | Path to the static files for the web interface. Can be absolute or relative to the config dir. If both `templates_path` and `static_files_path` are empty, the built-in web interface is disabled. |
| `openapi_path` | string | `openapi` | Path to the directory containing the OpenAPI schema and default renderer. Can be absolute or relative to the config dir. If empty, the OpenAPI schema and renderer are not served regardless of the `render_openapi` directive. |
| `web_root` | string | empty | Base URL for the web admin and client interfaces. If empty, resources are available at the root (`/`) URI. Must be an absolute URI or it will be ignored. |
| `certificate_file` | string | empty | Certificate for HTTPS. Can be absolute or relative to the config dir. |
| `certificate_key_file` | string | empty | Private key matching the above certificate. Can be absolute or relative to the config dir. If both certificate and key are provided, HTTPS can be enabled for the configured bindings. Certificate and key files can be reloaded on demand by sending a `SIGHUP` signal on Unix-based systems or a `paramchange` request to the running service on Windows. Certificates are also polled for changes every 8 hours. |
| `ca_certificates` | list of strings | empty | Root certificate authorities used to verify client certificates. |
| `ca_revocation_lists` | list of strings | empty | Revocation lists (one per root CA) to check if a client certificate has been revoked. Can be reloaded on demand via `SIGHUP` (Unix) or `paramchange` (Windows). |
| `signing_passphrase` | string | empty | Passphrase used to derive the signing key for JWT and CSRF tokens. If empty, a random signing key is generated each time SFTPGo starts. Consider rotating it periodically for added security. |
| `signing_passphrase_file` | string | empty | Path to a file containing the signing passphrase. Can be absolute or relative to the config dir. If not empty, takes precedence over `signing_passphrase`. |
| `token_validation` | integer | `0` | Defines how JWT tokens, cookies, and CSRF tokens are validated. By default, a token must be used by the same IP for which it was issued. `1` disables this requirement. `2` invalidates admin and user tokens issued before the last update. Flags can be combined. |
| `cookie_lifetime` | integer | `20` | Duration in minutes for WebAdmin and WebClient cookies. Cookies are automatically refreshed on user activity (maximum duration: 12 hours). Maximum allowed value: `720`. |
| `share_cookie_lifetime` | integer | `120` | Duration in minutes for public sharing cookies. Maximum allowed value: `720`. |
| `jwt_lifetime` | integer | `20` | Duration in minutes for REST API tokens. Maximum allowed value: `720`. |
| `max_upload_file_size` | integer | `0` | Maximum request body size, in bytes, for Web Client/API HTTP upload requests. `0` means no limit. |
| `hide_support_link` | boolean | `false` | If set, the link to the [sponsors section](https://github.com/drakkan/sftpgo?tab=readme-ov-file#sponsors){:target="_blank"} is not displayed on the setup screen. |

#### bindings

Each binding is a struct with the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `port` | integer | `8080` | Port used for serving HTTP requests. |
| `address` | string | empty | Leave blank to listen on all available network interfaces. On *NIX systems, you can specify an absolute path to listen on a Unix-domain socket. |
| `enable_web_admin` | boolean | `true` | Set to `false` to disable the built-in web admin for this binding. Requires `templates_path` and `static_files_path` to be defined. |
| `enable_web_client` | boolean | `true` | Set to `false` to disable the built-in web client for this binding. Requires `templates_path` and `static_files_path` to be defined. |
| `enable_rest_api` | boolean | `true` | Set to `false` to disable the REST API for this binding. |
| `enabled_login_methods` | integer | `0` | :warning: DEPRECATED: use `disabled_login_methods`. Defines available login methods for the WebAdmin and WebClient UIs. `0` means any configured method (username/password and OIDC if enabled). `1` OIDC for WebAdmin. `2` OIDC for WebClient. `4` login form for WebAdmin. `8` login form for WebClient. Flags can be combined. |
| `disabled_login_methods` | integer | `0` | Defines disabled login methods. `0` means all methods are enabled. `1` OIDC for WebAdmin. `2` OIDC for WebClient. `4` login form for WebAdmin. `8` login form for WebClient. `16` admin token endpoint for REST API. `32` user token endpoint for REST API. `64` admin API key login. `128` user API key login. Flags can be combined (e.g., `252` allows only OIDC login for both WebClient and WebAdmin). |
| `enable_https` | boolean | `false` | Set to `true` and provide both a certificate and a key file to enable HTTPS for this binding. |
| `certificate_file` | string | empty | Binding-specific TLS certificate. Can be absolute or relative to the config dir. |
| `certificate_key_file` | string | empty | Binding-specific private key matching the above certificate. Can be absolute or relative to the config dir. If not set, the global ones are used (if any). |
| `min_tls_version` | integer | `12` | Minimum TLS version. `10` TLS 1.0, `11` TLS 1.1, `12` TLS 1.2, `13` TLS 1.3. |
| `client_auth_type` | integer | `0` | Set to `1` to require client certificate authentication in addition to JWT/Web authentication. At least one certificate authority must be defined. |
| `tls_cipher_suites` | list of strings | empty | Supported cipher suites for TLS 1.2 and below. If empty, a default list of secure cipher suites is used. TLS 1.3 cipher suites are not configurable. See supported [cipher suite names](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Invalid names are silently ignored. Order matters (listed first = preferred). |
| `tls_protocols` | list of strings | `http/1.1`, `h2` | HTTPS protocols in preference order. Supported values: `http/1.1`, `h2`. |
| `proxy_mode` | integer | `0` | Set to `1` to use the proxy protocol configuration defined in the `common` section instead of the proxy header configuration. |
| `proxy_allowed` | list of strings | empty | IP addresses and ranges allowed to set client IP proxy headers (`X-Forwarded-For`, `X-Real-IP`, etc.). Headers set by connections from addresses not in this list are silently ignored. |
| `client_ip_proxy_header` | string | empty | Allowed client IP proxy header (e.g., `X-Forwarded-For`, `X-Real-IP`). |
| `client_ip_header_depth` | integer | `0` | For multi-value headers like `X-Forwarded-For`, defines which IP to trust counting from the right. `0` uses the rightmost IP, `1` uses the second from right, etc. Set to `-1` to trust the leftmost IP. :warning: Using `-1` may have security implications and should only be used if your proxy has appropriate controls to prevent spoofed headers. |
| `hide_login_url` | integer | `0` | Controls the cross-link between admin and client login pages. `0` shows both links. `1` hides the web client link on the admin login page. `2` hides the web admin link on the client login page. Flags can be combined (e.g., `3` hides both). |
| `render_openapi` | boolean | `true` | Set to `false` to disable serving of the OpenAPI schema and renderer. |
| `languages` | list of strings | `en` | Supported values: `en`, `it`, `de`, `fr`, `es`, `zh-CN`. |

#### oidc

OpenID Connect configuration. OIDC integration allows you to map your identity provider users to SFTPGo users, enabling login to the WebClient and WebAdmin interfaces via your identity provider.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `config_url` | string | empty | Service identifier. If defined, SFTPGo appends `/.well-known/openid-configuration` to this URL and attempts to retrieve the provider configuration on startup. SFTPGo refuses to start if the URL is unreachable. |
| `client_id` | string | empty | Application ID. |
| `client_secret` | string | empty | Application secret. |
| `client_secret_file` | string | empty | Path to a file containing the application secret. Can be absolute or relative to the config dir. If not empty, takes precedence over `client_secret`. |
| `redirect_base_url` | string | empty | Base URL to redirect to after OpenID authentication. The suffix `/web/oidc/redirect` is appended automatically (including `web_root` if configured). |
| `username_field` | string | empty | ID token claims field to map to the SFTPGo username. |
| `scopes` | list of strings | `openid`, `profile`, `email` | OAuth scopes to request. The `openid` scope is mandatory. |
| `role_field` | string | empty | Optional ID token claim used to determine the SFTPGo role. If the claim value is `admin`, the authenticated user is mapped to the SFTPGo admin role. Dot notation can be used for nested claims. Only required if you want to use OpenID for the WebAdmin UI. |
| `implicit_roles` | boolean | `false` | If set, `role_field` is ignored and the SFTPGo role is inferred from the login link used. |
| `custom_fields` | list of strings | empty | Custom token claim fields to pass to the pre-login hook. |
| `insecure_skip_signature_check` | boolean | `false` | :warning: Skips JWT signature validation. Intended for special cases where providers (e.g., Azure) use the `none` algorithm. Skipping validation can cause security issues. |
| `debug` | boolean | `false` | If set, received ID tokens are logged at debug level. |

#### security

Security headers added to HTTP responses and host restriction settings.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `enabled` | boolean | `false` | Set to `true` to enable security configurations. |
| `allowed_hosts` | list of strings | empty | Fully qualified domain names that are allowed. An empty list allows all host names. |
| `allowed_hosts_are_regex` | boolean | `false` | Whether the provided allowed hosts contain valid regular expressions. |
| `hosts_proxy_headers` | list of strings | empty | Header keys that may hold a proxied hostname value (e.g., `X-Forwarded-Host`). |
| `https_redirect` | boolean | `false` | Set to `true` to redirect HTTP requests to HTTPS. If redirecting from port `80` and using built-in ACME with `HTTP-01` challenge, use the webroot method and set the ACME web root to a path writable by SFTPGo. |
| `https_host` | string | empty | Host name used for HTTPS redirects. If blank, the same host is used. For example, with `https_host` set to `www.example.com`, a request for `http://127.0.0.1/web/client/login` redirects to `https://www.example.com/web/client/login`. |
| `https_proxy_headers` | list of structs | empty | Each struct has `key` and `value` fields. Defines headers that indicate a valid HTTPS request (e.g., `key`: `X-Forwarded-Proto`, `value`: `https`). |
| `sts_seconds` | integer | `0` | Max-age of the `Strict-Transport-Security` header. Included for HTTPS responses or HTTP requests with a defined HTTPS proxy header. `0` means the header is not included. |
| `sts_include_subdomains` | boolean | `false` | Appends `includeSubdomains` to the `Strict-Transport-Security` header. |
| `sts_preload` | boolean | `false` | Appends `preload` to the `Strict-Transport-Security` header. |
| `content_type_nosniff` | boolean | `false` | Adds the `X-Content-Type-Options` header with value `nosniff`. |
| `content_security_policy` | string | empty | Value for the `Content-Security-Policy` header. |
| `permissions_policy` | string | empty | Value for the `Permissions-Policy` header. |
| `cross_origin_opener_policy` | string | empty | Value for the `Cross-Origin-Opener-Policy` header. |
| `cross_origin_resource_policy` | string | empty | Value for the `Cross-Origin-Resource-Policy` header. |
| `cross_origin_embedder_policy` | string | empty | Value for the `Cross-Origin-Embedder-Policy` header. |
| `referrer_policy` | string | empty | Value for the `Referrer-Policy` header. |
| `cache_control` | string | empty | Value for the `Cache-Control` header. Set to `private` to disable caching for dynamic pages. |

#### branding

Customizations to suit your brand. Contains `web_admin` and `web_client` sub-structs that define customizations for the WebAdmin and WebClient UIs respectively. Each sub-struct has the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `name` | string | empty | UI name. |
| `short_name` | string | empty | Short name displayed next to the logo image and on the login page. |
| `favicon_path` | string | empty | Path to the favicon relative to `static_files_path` (e.g., `/branding/favicon.png`). |
| `logo_path` | string | empty | Path to your logo relative to `static_files_path`. Preferred image size: 256x256 pixels. |
| `disclaimer_name` | string | empty | Name for your optional disclaimer. |
| `disclaimer_path` | string | empty | Path to the HTML disclaimer page relative to `static_files_path`, or an absolute URL (http or https). |
| `default_css` | list of strings | empty | Paths to custom CSS files (relative to `static_files_path`) that replace the default CSS. |
| `extra_css` | list of strings | empty | Paths to additional CSS files (relative to `static_files_path`). |

#### cors

CORS configuration. SFTPGo uses [Go CORS handler](https://github.com/rs/cors){:target="_blank"}; refer to upstream documentation for field semantics and default values.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `enabled` | boolean | `false` | Set to `true` to enable CORS. |
| `allowed_origins` | list of strings | empty | Allowed origins. |
| `allowed_methods` | list of strings | empty | Allowed HTTP methods. |
| `allowed_headers` | list of strings | empty | Allowed request headers. |
| `exposed_headers` | list of strings | empty | Headers exposed to the browser. |
| `allow_credentials` | boolean | `false` | Whether credentials (cookies, authorization headers) are allowed. |
| `max_age` | integer | `0` | Max age (in seconds) for preflight request caching. |
| `options_passthrough` | boolean | `false` | Whether OPTIONS requests are passed through to the handler. |
| `options_success_status` | integer | `0` | HTTP status code for successful OPTIONS requests. |
| `allow_private_network` | boolean | `false` | Whether private network access is allowed. |

#### setup

Configuration for the initial setup screen.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `installation_code` | string | empty | If set, this code is required when creating the first admin account. Read at SFTPGo startup, not at runtime. :warning: This is not a license key; it prevents unauthorized users from creating an admin account via the setup screen. |
| `installation_code_hint` | string | `Installation code` | Description for the installation code input field. |

## Telemetry

Configuration parameters for the `telemetry` section.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `bind_port` | integer | `0` | The port used for serving HTTP requests. Set to `0` to disable the HTTP server. |
| `bind_address` | string | `127.0.0.1` | Leave blank to listen on all available network interfaces. On *NIX systems you can specify an absolute path to listen on a Unix-domain socket. |
| `enable_profiler` | boolean | `false` | Enable the built-in profiler. |
| `auth_user_file` | string | empty | Path to a file used to store usernames and passwords for basic authentication. Can be absolute or relative to the config dir. The file format must conform to the one generated by the Apache `htpasswd` tool. Supported password formats are bcrypt (`$2y$` prefix) and md5 crypt (`$apr1$` prefix). If empty, HTTP authentication is disabled. Authentication is always disabled for the `/healthz` endpoint. |
| `certificate_file` | string | empty | Certificate for HTTPS. Can be absolute or relative to the config dir. |
| `certificate_key_file` | string | empty | Private key matching the above certificate. Can be absolute or relative to the config dir. If both the certificate and the private key are provided, the server will expect HTTPS connections. Certificate and key files can be reloaded on demand by sending a `SIGHUP` signal on Unix-based systems and a `paramchange` request to the running service on Windows. |
| `min_tls_version` | integer | `12` | Minimum TLS version to enable. `10` = TLS 1.0, `11` = TLS 1.1, `12` = TLS 1.2, `13` = TLS 1.3. |
| `tls_cipher_suites` | list of strings | empty | Supported cipher suites for TLS 1.2 and below. If empty, a default list of secure cipher suites is used, with preference order based on hardware performance. TLS 1.3 cipher suites are not configurable. See supported [cipher suite names](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Invalid names are silently ignored. Order matters: ciphers listed first are preferred. |
| `tls_protocols` | list of strings | `http/1.1`, `h2` | HTTPS protocols in preference order. Supported values: `http/1.1`, `h2`. |

The telemetry server exposes the following endpoints:

- `/healthz` — health information (for health checks)
- `/metrics` — Prometheus metrics
- `/debug/pprof` — profiling (when `enable_profiler` is enabled), [more details](profiling.md)

## HTTP clients

HTTP clients are used for executing hooks. Some hooks use a retryable HTTP client; for these you can configure the time between retries and the number of retries. Check the hook-specific documentation to understand which hooks use a retryable HTTP client.

Configuration parameters for the `http` section.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `timeout` | float | `20` | Time limit in seconds for requests. For retryable requests, this is the timeout for a single attempt. |
| `retry_wait_min` | integer | `2` | Minimum waiting time between retry attempts, in seconds. |
| `retry_wait_max` | integer | `30` | Maximum waiting time between retry attempts, in seconds. The backoff algorithm performs exponential backoff based on the attempt number, bounded by the minimum and maximum durations. |
| `retry_max` | integer | `3` | Maximum number of retries if the first request fails. |
| `ca_certificates` | list of strings | empty | Paths to extra CA certificates to trust. Paths can be absolute or relative to the config dir. This is a convenient way to use self-signed certificates without disabling TLS verification. |
| `certificates` | list of structs | empty | Client certificates for mutual TLS. See [certificates](#http-client-certificates) below. |
| `skip_tls_verify` | boolean | `false` | If enabled, the HTTP client accepts any TLS certificate presented by the server and any host name in that certificate. :warning: In this mode, TLS is susceptible to man-in-the-middle attacks. Use only for testing. |
| `headers` | list of structs | empty | HTTP headers to add to each hook request. See [headers](#http-client-headers) below. |

#### HTTP client certificates

Each entry in the `certificates` list is a struct with the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `cert` | string | empty | Path to the certificate file. Can be absolute or relative to the config dir. |
| `key` | string | empty | Path to the key file. Can be absolute or relative to the config dir. |

#### HTTP client headers

Each entry in the `headers` list is a struct with the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `key` | string | empty | Header name. The header is silently ignored if `key` or `value` are empty. |
| `value` | string | empty | Header value. The header is silently ignored if `key` or `value` are empty. |
| `url` | string | empty | Optional. If not empty, the header is added only when the request URL starts with this value. |

## Commands

External commands are used for executing hooks.

Configuration parameters for the `command` section.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `timeout` | integer | `30` | Time limit in seconds for executing external commands. Valid range: `1`–`300`. |
| `env` | list of strings | empty | Environment variables to pass to all external commands. Global environment variables are cleared for security reasons; you must explicitly set any needed variables such as `PATH`. Each entry uses the format `key=value`. Do not use variables prefixed with `SFTPGO_` to avoid conflicts with environment variables that SFTPGo hooks can set. |
| `commands` | list of structs | empty | Per-command configuration overrides. See [command overrides](#command-overrides) below. |

#### Command overrides

Each entry in the `commands` list is a struct with the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `path` | string | empty | Command path as defined in the hook configuration. |
| `timeout` | integer | — | Overrides the global timeout for this command, if set. |
| `env` | list of strings | empty | Additional environment variables appended to the global ones defined for all commands. |
| `args` | list of strings | empty | Arguments to pass to the command identified by `path`. |
| `hook` | string | empty | If not empty, this configuration applies only to the specified hook name. Supported hook names: `fs_actions`, `provider_actions`, `startup`, `post_connect`, `post_disconnect`, `check_password`, `pre_login`, `post_login`, `external_auth`, `keyboard_interactive`. |

## KMS

Configuration parameters for the `kms` section.

#### secrets

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `url` | string | empty | URI to the KMS service. |
| `master_key` | string | empty | Master encryption key as a string. |
| `master_key_path` | string | empty | Absolute path to a file containing the master encryption key. If not empty, takes precedence over `master_key`. |

## Multi-factor authentication

Configuration parameters for the `mfa` section.

#### TOTP

The `totp` field is a list of structs that define settings for time-based one-time passwords (RFC 6238). Each struct has the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `name` | string | `Default` | Unique configuration name. This name should not be changed if there are users or admins using the configuration. The name is not visible to authentication apps. |
| `issuer` | string | `SFTPGo` | Name of the issuing organization/company. |
| `algo` | string | `sha1` | Algorithm to use for HMAC. Supported algorithms: `sha1`, `sha256`, `sha512`. :information_source: Google Authenticator on iPhone currently appears to support only `sha1` — check compatibility with your target apps/devices before choosing a different algorithm. You can define multiple configurations (e.g., one with `sha256` or `sha512` and another with `sha1`) and instruct users to pick the appropriate one. The algorithm should not be changed if there are users or admins using the configuration. |

## SMTP

SMTP configuration enables SFTPGo email sending capabilities.

Configuration parameters for the `smtp` section.

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `host` | string | empty | SMTP server hostname. Leave empty to disable email sending capabilities. |
| `port` | integer | `587` | SMTP server port. |
| `from` | string | empty | From address, for example `SFTPGo <sftpgo@example.com>`. Many SMTP servers reject emails without a `From` header; if not set, SFTPGo will try to use the username as a fallback, which may or may not be appropriate. |
| `user` | string | empty | SMTP username. |
| `password` | string | empty | SMTP password. Leaving both username and password empty disables SMTP authentication. |
| `auth_type` | integer | `0` | Authentication type. `0` = Plain, `1` = Login, `2` = CRAM-MD5, `3` = XOAUTH2. |
| `encryption` | integer | `0` | Encryption mode. `0` = none, `1` = TLS, `2` = STARTTLS. |
| `domain` | string | empty | Domain to use for the `HELO` command. If empty, `localhost` is used. |
| `templates_path` | string | `templates` | Path to email templates. Can be absolute or relative to the config dir. Templates are searched within a subdirectory named `email` in the specified path. You can customize email templates by specifying an alternate path and placing your custom templates there. |
| `debug` | integer | `0` | Set to `1` to enable SMTP debug logging. |
| `oauth2` | struct | — | OAuth2-related configuration. See [SMTP OAuth2](#smtp-oauth2) below. |

#### SMTP OAuth2

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `provider` | integer | `0` | OAuth2 provider. `0` = Google, `1` = Microsoft. |
| `tenant` | string | empty | Azure Active Directory tenant for the Microsoft provider. Typical values are `common`, `organizations`, `consumers`, or a tenant identifier. If empty, `common` is used. |
| `client_id` | string | empty | OAuth2 client ID. |
| `client_secret` | string | empty | OAuth2 client secret. |
| `refresh_token` | string | empty | OAuth2 refresh token. |

## Plugins

:warning: The plugin system is experimental. Configuration parameters and interfaces may change in a backward-incompatible way in future releases.

The `plugins` section defines a list of external plugins. The default value is an empty list (no plugins enabled). Each plugin is configured using a struct with the following fields:

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `type` | string | — | Plugin type. Supported types: `notifier`, `kms`, `auth`, `eventsearcher`, `ipfilter`. |
| `notifier_options` | struct | — | Options for notifier plugins. See [notifier options](#notifier-options) below. |
| `kms_options` | struct | — | Options for KMS plugins. See [KMS options](#kms-plugin-options) below. |
| `auth_options` | struct | — | Options for auth plugins. See [auth options](#auth-plugin-options) below. |
| `cmd` | string | — | Path to the plugin executable. |
| `args` | list of strings | empty | Optional arguments to pass to the plugin executable. |
| `sha256sum` | string | empty | SHA256 checksum for the plugin executable. If not empty, it is used to verify the integrity of the executable. |
| `auto_mtls` | boolean | `false` | If enabled, the client and server automatically negotiate mutual TLS for transport authentication. This ensures that only the original client is allowed to connect to the server (all other connections are rejected), and the client refuses to connect to any server that is not the original instance started by the client. |
| `env_prefix` | string | empty | Prefix for environment variables to pass from the SFTPGo process environment to the plugin. Set to `none` to pass no environment variables, or `*` to pass all. If empty, the prefix is derived from the plugin name in uppercase with `-` replaced by `_` and a trailing `_`. For example, if the plugin name is `sftpgo-plugin-eventsearch`, the prefix will be `SFTPGO_PLUGIN_EVENTSEARCH_`. |
| `env_vars` | list of strings | empty | Additional environment variable names to pass from the SFTPGo process environment to the plugin. |

#### Notifier options

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `fs_events` | list of strings | empty | Filesystem events to notify. Supported values: `download`, `upload`, `first-upload`, `first-download`, `delete`, `rename`, `mkdir`, `rmdir`, `copy`, `ssh_cmd`. |
| `provider_events` | list of strings | empty | Provider events to notify. Supported values: `add`, `update`, `delete`. |
| `provider_objects` | list of strings | empty | Provider objects to notify. Supported values: `user`, `folder`, `group`, `admin`, `api_key`, `share`, `event_action`, `event_rule`, `role`, `ip_list_entry`, `configs`. |
| `log_events` | list of integers | empty | Log events to notify. `1` = Login failed, `2` = Login with non-existent user, `3` = No login tried, `4` = Algorithm negotiation failed, `5` = Login succeeded. |
| `retry_max_time` | integer | `0` | Maximum number of seconds an event can be late. SFTPGo timestamps each event and queues any events that the plugin fails to handle. Events older than this limit are discarded. SFTPGo retries queued events every 30 seconds. `0` means no retry. |
| `retry_queue_max_size` | integer | `0` | Maximum number of events the internal queue can hold. Once the queue is full, events that cannot be sent to the plugin are discarded. `0` means no limit. |

#### KMS plugin options

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `scheme` | string | empty | KMS scheme. Supported schemes: `awskms`, `gcpkms`, `hashivault`, `azurekeyvault`. |
| `encrypted_status` | string | empty | Encrypted status for a KMS secret. Supported statuses: `AWS`, `GCP`, `VaultTransit`, `AzureKeyVault`. |

#### Auth plugin options

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `scope` | integer | — | Authentication scope. `1` = passwords only, `2` = public keys only, `4` = keyboard interactive only, `8` = TLS certificate. Flags can be combined (e.g., `6` = public keys and keyboard interactive). :warning: The scope must be explicit; `0` is not a valid option. |
