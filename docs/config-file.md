# Configuration file

The default configuration file is [sftpgo.json](https://github.com/drakkan/sftpgo/blob/main/sftpgo.json){:target="_blank"} and is divided in several sections.

## Common

Supported configuration parameters for the `common` section:

- `idle_timeout`, integer. Time in minutes after which an idle client will be disconnected. 0 means disabled. Default: 15
- `upload_mode` integer. `0` means standard: the files are uploaded directly to the requested path. `1` means atomic: files are uploaded to a temporary path and renamed to the requested path when the client ends the upload. Atomic mode avoids problems such as a web server that serves partial files when the files are being uploaded. In atomic mode, if there is an upload error, the temporary file is deleted and so the requested upload path will not contain a partial file. `2` means atomic with resume support: same as atomic but if there is an upload error, the temporary file is renamed to the requested path and not deleted. This way, a client can reconnect and resume the upload. `4` means files for S3 backend are stored even if a client-side upload error is detected. `8` means files for Google Cloud Storage backend are stored even if a client-side upload error is detected. `16` means files for Azure Blob backend are stored even if a client-side upload error is detected. Ignored for SFTP backend if buffering is enabled. The flags can be combined, if you provide both `1` and `2`, `2` will be used. Default: `0`
- `actions`, struct. It contains the command to execute and/or the HTTP URL to notify and the trigger conditions. See [Custom Actions](custom-actions.md) for more details
  - `execute_on`, list of strings. Valid values are `pre-download`, `download`, `first-download`, `pre-upload`, `upload`, `first-upload`, `pre-delete`, `delete`, `rename`, `mkdir`, `rmdir`, `ssh_cmd`, `copy`. Leave empty to disable actions.
  - `execute_sync`, list of strings. Actions, defined in the `execute_on` list above, to be performed synchronously. The `pre-*` actions are always executed synchronously while the other ones are asynchronous. Executing an action synchronously means that SFTPGo will not return a result code to the client (which is waiting for it) until your hook have completed its execution. Leave empty to execute only the defined `pre-*` hook synchronously
  - `hook`, string. Absolute path to the command to execute or HTTP URL to notify.
- `setstat_mode`, integer. 0 means "normal mode": requests for changing permissions, owner/group and access/modification times are executed. 1 means "ignore mode": requests for changing permissions, owner/group and access/modification times are silently ignored. 2 means "ignore mode if not supported": requests for changing permissions and owner/group are silently ignored for cloud filesystems and executed for local/SFTP filesystem.
- `rename_mode`, integer. By default (`0`), renaming of non-empty directories is not allowed for cloud storage providers (S3, GCS, Azure Blob). Set to `1` to enable recursive renames for these providers, they may be slow, there is no atomic rename API like for local filesystem, so SFTPGo will recursively list the directory contents and do a rename for each entry (partial renaming and incorrect disk quota updates are possible in error cases). Default `0`.
- `resume_max_size`, integer. defines the maximum size allowed, in bytes, to resume uploads on storage backends with immutable objects. By default, resuming uploads is not allowed for cloud storage providers (S3, GCS, Azure Blob) because SFTPGo must rewrite the entire file. Set to a value greater than 0 to allow resuming uploads of files smaller than or equal to the defined size. Please note that uploads for these backends are still atomic, the client must intentionally upload a portion of the target file and then resume uploading.. Default `0`.
- `temp_path`, string. Defines the path for temporary files such as those used for atomic uploads or file pipes. If you set this option you must make sure that the defined path exists, is accessible for writing by the user running SFTPGo, and is on the same filesystem as the users home directories otherwise the renaming for atomic uploads will become a copy and therefore may take a long time. The temporary files are not namespaced. The default is generally fine. Leave empty for the default.
- `proxy_protocol`, integer. Support for [HAProxy PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt){:target="_blank"}. If you are running SFTPGo behind a proxy server such as HAProxy, AWS ELB or NGINX, and your proxy is not able to preserve the IP address of clients, you can enable the proxy protocol. It provides a convenient way to safely transport connection information such as a client's address across multiple layers of NAT or TCP proxies to get the real client IP address instead of the proxy IP. Both protocol versions 1 and 2 are supported. If the proxy protocol is enabled in SFTPGo then you have to enable the protocol in your proxy configuration too. For example, for HAProxy, add `send-proxy` or `send-proxy-v2` to each server configuration line. The PROXY protocol is supported for SSH/SFTP and FTP/S. The following modes are supported:
  - 0, disabled
  - 1, enabled. If the upstream IP is not allowed to send a proxy header, the header will be ignored. Using this mode does not mean that we can accept connections with and without the proxy header. We always try to read the proxy header and we ignore it if the upstream IP is not allowed to send a proxy header. Set `proxy_skipped` if you want to allow some IPs/networks to connect without sending a proxy header and without SFTPGo trying to read it
  - 2, required. If the upstream IP is not allowed to send a proxy header, the connection will be rejected if a proxy header is found. We always try to read the proxy header. Set `proxy_skipped` if you want to allow some IPs/networks to connect without sending a proxy header and without SFTPGo trying to read it
- `proxy_allowed`, list of IP addresses and IP ranges allowed to send the proxy header:
  - If `proxy_protocol` is set to 1 and we receive a proxy header from an IP that is not in the list then the connection will be accepted and the header will be ignored
  - If `proxy_protocol` is set to 2 and we receive a proxy header from an IP that is not in the list then the connection will be rejected
- `proxy_skipped`, list of IP address and IP ranges for which not to read the proxy header
- `startup_hook`, string. Absolute path to an external program or an HTTP URL to invoke as soon as SFTPGo starts. If you define an HTTP URL it will be invoked using a `GET` request. Please note that SFTPGo services may not yet be available when this hook is run. Leave empty do disable
- `post_connect_hook`, string. Absolute path to the command to execute or HTTP URL to notify. See [Post-connect hook](post-connect-hook.md) for more details. Leave empty to disable
- `post_disconnect_hook`, string. Absolute path to the command to execute or HTTP URL to notify. See [Post-disconnect hook](post-disconnect-hook.md) for more details. Leave empty to disable
- `data_retention_hook`, string. Absolute path to the command to execute or HTTP URL to notify. See [Data retention hook](data-retention-hook.md) for more details. Leave empty to disable
- `max_total_connections`, integer. Maximum number of concurrent client connections. 0 means unlimited. Default: `0`.
- `max_per_host_connections`, integer.  Maximum number of concurrent client connections from the same host (IP). If the defender is enabled, exceeding this limit will generate `score_limit_exceeded` events and thus hosts that repeatedly exceed the max allowed connections can be automatically blocked. 0 means unlimited. Default: `20`.
- `allowlist_status`, integer. Set to `1` to enable the allow list. The allow list can be populated using the WebAdmin or the REST API. If enabled, only the listed IPs/networks can access the configured services, all other client connections will be dropped before they even try to authenticate. Ensure to populate your allow list before enabling this setting. In multi-nodes setups, the list entries propagation between nodes may take some minutes. Default: `0`.
- `allow_self_connections`, integer. Allow users on this instance to use other users/virtual folders on this instance as storage backend. Enable this setting if you know what you are doing. Set to `1` to enable. Default: `0`.
- `umask`, string. Set the file mode creation mask, for example `002`. Leave blank to use the system umask. Supported on *NIX platforms. Default: blank.
- `server_version`, string. Allow some degree of customization for the advertised software version. Set to `short` to hide the SFTPGo version number, if different from `short` the default will be used. Default: blank.
- `tz`, string. Defines the time zone to use for the EventManager scheduler and to control time-based access restrictions. Set to `local` to use the server's local time, otherwise UTC will be used. Default: blank.
- `metadata`, struct containing the configuration for managing the Cloud Storage backends metadata.
  - `read`, integer. Set to `1` to read metadata before downloading files from Cloud Storage backends and making them available in notification events. Default: `0`.
- `defender`, struct containing the defender configuration. See [Defender](defender.md) for more details.
  - `enabled`, boolean. Default `false`.
  - `driver`, string. Supported drivers are `memory` and `provider`. The `provider` driver will use the configured data provider to store defender events and it is supported for `MySQL`, `PostgreSQL` and `CockroachDB` data providers. Using the `provider` driver you can share the defender events among multiple SFTPGO instances. For a single instance the `memory` driver will be much faster. Default: `memory`.
  - `ban_time`, integer. Ban time in minutes. Default: `30`.
  - `ban_time_increment`, integer. Ban time increment, as a percentage, if a banned host tries to connect again. Default: `50`.
  - `threshold`, integer. Threshold value for banning a client. Default: `15`.
  - `score_invalid`, integer. Score for invalid login attempts, eg. non-existent user accounts. Default: `2`.
  - `score_valid`, integer. Score for valid login attempts, eg. user accounts that exist. Default: `1`.
  - `score_limit_exceeded`, integer. Score for hosts that exceeded the configured rate limits or the maximum, per-host, allowed connections. Default: `3`.
  - `score_no_auth`, defines the score for clients disconnected without any authentication attempt. Default: `0`.
  - `observation_time`, integer. Defines the time window, in minutes, for tracking client errors. A host is banned if it has exceeded the defined threshold during the last observation time minutes. Default: `30`.
  - `entries_soft_limit`, integer. Ignored for `provider` driver. Default: `100`.
  - `entries_hard_limit`, integer. The number of banned IPs and host scores kept in memory will vary between the soft and hard limit for `memory` driver. If you use the `provider` driver, this setting will limit the number of entries to return when you ask for the entire host list from the defender. Default: `150`.
  - `login_delay`, struct containing the configuration to impose a delay between login attempts:
    - `success`, integer. Defines the number of milliseconds to pause prior to allowing a successful login. `0` means disabled. Default: `0`.
    - `password_failed`, integer. Defines the number of milliseconds to pause prior to reporting a failed password/interactive login. `0` means disabled. Default: `1000`.
- `rate_limiters`, list of structs containing the rate limiters configuration. Take a look [here](rate-limiting.md) for more details. Each struct has the following fields:
  - `average`, integer. Average defines the maximum rate allowed. 0 means disabled. Default: 0
  - `period`, integer. Period defines the period as milliseconds. The rate is actually defined by dividing average by period Default: 1000 (1 second).
  - `burst`, integer. Burst defines the maximum number of requests allowed to go through in the same arbitrarily small period of time. Default: 1
  - `type`, integer. 1 means a global rate limiter, independent from the source host. 2 means a per-ip rate limiter. Default: 2
  - `protocols`, list of strings. Available protocols are `SSH`, `FTP`, `DAV`, `HTTP`. By default all supported protocols are enabled
  - `generate_defender_events`, boolean. If `true`, the defender is enabled, and this is not a global rate limiter, a new defender event will be generated each time the configured limit is exceeded. Default `false`
  - `entries_soft_limit`, integer.
  - `entries_hard_limit`, integer. The number of per-ip rate limiters kept in memory will vary between the soft and hard limit
- `event_manager`, struct containing the configuration for the EventManager
  - `enabled_commands`, list of strings. Absolute path to system commands that can be executed through Event Manager. An empty list means that no commands are allowed to be executed. :warning: Allowing system command could pose a security risk. Default: empty

## ACME

Automatic Certificate Management Environment (ACME) protocol configuration. To obtain the certificates the first time you have to configure the ACME protocol and execute the `sftpgo acme run` command or use the WebAdmin UI. The SFTPGo service will take care of the automatic renewal of certificates for the configured domains.

Supported configuration parameters for the `acme` section:

- `domains`, list of domains for which to obtain certificates. If a single certificate is to be valid for multiple domains specify the names separated by commas or spaces, for example: `example.com,www.example.com` or `example.com www.example.com`. An empty list means that ACME protocol is disabled. Default: empty.
- `email`, string. Email used for registration and recovery contact. Default: empty.
- `key_type`, string. Key type to use for private keys. Supported values: `2048` (RSA 2048), `3072` (RSA 3072), `4096` (RSA 4096), `8192` (RSA 8192), `P256` (EC 256), `P384` (EC 384). Default: `4096`
- `certs_path`, string. Directory, absolute or relative to the configuration directory, to use for storing certificates and related data.
- `ca_endpoint`, string. Default: `https://acme-v02.api.letsencrypt.org/directory`.
- `renew_days`, integer. The number of days left on a certificate to renew it. Default: `30`.
- `http01_challenge`, configuration for `HTTP-01` challenge type, the following fields are supported:
  - `port`, integer. This challenge is expected to run on port `80`. If you set a port other than `80` you have to proxy the path `/.well-known/acme-challenge` from the port `80` to the configured port. Default: `80`.
  - `proxy_header`, string. Validate against this HTTP header when solving HTTP based challenges behind a reverse proxy. Empty means `Host`. Default: empty.
  - `webroot`, string. Set the absolute path to the webroot folder to use for HTTP based challenges to write directly in a file in `.well-known/acme-challenge`. Setting a `webroot` disables the built-in server (the `port` setting is ignored) and expects the given directory to be publicly served, on port `80`, with access to `.well-known/acme-challenge`. If `webroot` is empty and `port` is `0` the `HTTP-01` challenge is disabled. Default: empty.
- `tls_alpn01_challenge`, configuration for `TLS-ALPN-01` challenge type, the following fields are supported:
  - `port`, integer. This challenge is expected to run on port `443`. `0` means `TLS-ALPN-01` is disabled. Default: `0`.

## SSH/SFTP server

Supported configuration parameters for the `sftpd` section:

- `bindings`, list of structs. Each struct has the following fields:
  - `port`, integer. The port used for serving SFTP requests. 0 means disabled. Default: 2022
  - `address`, string. Leave blank to listen on all available network interfaces. Default: ""
  - `apply_proxy_config`, boolean. If enabled the common proxy configuration, if any, will be applied. Default `true`
- `max_auth_tries` integer. Maximum number of authentication attempts permitted per connection. If set to a negative number, the number of attempts is unlimited. If set to zero, the number of attempts is limited to 6.
- `host_keys`, list of strings. It contains the daemon's private host keys. Each host key can be defined as a path relative to the configuration directory or an absolute one. If empty, the daemon will search or try to generate `id_rsa`, `id_ecdsa` and `id_ed25519` keys inside the configuration directory. If you configure absolute paths to files named `id_rsa`, `id_ecdsa` and/or `id_ed25519` then SFTPGo will try to generate these keys using the default settings.
- `host_certificates`, list of strings. Public host certificates. Each certificate can be defined as a path relative to the configuration directory or an absolute one. Certificate's public key must match a private host key otherwise it will be silently ignored. Default: empty.
- `host_key_algorithms`, list of strings. Public key algorithms that the server will accept for host key authentication. The supported values are: `rsa-sha2-512-cert-v01@openssh.com`, `rsa-sha2-256-cert-v01@openssh.com`, `ssh-rsa-cert-v01@openssh.com`, `ssh-dss-cert-v01@openssh.com`, `ecdsa-sha2-nistp256-cert-v01@openssh.com`, `ecdsa-sha2-nistp384-cert-v01@openssh.com`, `ecdsa-sha2-nistp521-cert-v01@openssh.com`, `ssh-ed25519-cert-v01@openssh.com`, `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `rsa-sha2-512`, `rsa-sha2-256`, `ssh-rsa`, `ssh-dss`, `ssh-ed25519`. Certificate algorithms are listed for backward compatibility purposes only, they are not used. Default values: `rsa-sha2-512`, `rsa-sha2-256`, `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `ssh-ed25519`.
- `kex_algorithms`, list of strings. Available KEX (Key Exchange) algorithms in preference order. Leave empty to use default values. The supported values are: `curve25519-sha256` (will also enable the alias `curve25519-sha256@libssh.org`), `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`, `diffie-hellman-group14-sha256`, `diffie-hellman-group16-sha512`, `diffie-hellman-group14-sha1`, `diffie-hellman-group1-sha1`, `diffie-hellman-group-exchange-sha256`, `diffie-hellman-group-exchange-sha1`. Default values: `curve25519-sha256`, `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`, `diffie-hellman-group14-sha256`,  `diffie-hellman-group-exchange-sha256`. SHA512 based KEXs are disabled by default because they are slow.
- `min_dh_group_exchange_key_size`, integer. Defines the minimum key size to allow for the key exchanges when using diffie-ellman-group-exchange-sha1 or sha256 key exchange algorithms. Allowed values: `2048`, `3072`. Default: `2048`.
- `ciphers`, list of strings. Allowed ciphers in preference order. Leave empty to use default values. The supported values are: `aes128-gcm@openssh.com`, `aes256-gcm@openssh.com`, `chacha20-poly1305@openssh.com`, `aes128-ctr`, `aes192-ctr`, `aes256-ctr`, `aes128-cbc`, `aes192-cbc`, `aes256-cbc`, `3des-cbc`, `arcfour256`, `arcfour128`, `arcfour`. Default values: `aes128-gcm@openssh.com`, `aes256-gcm@openssh.com`, `chacha20-poly1305@openssh.com`, `aes128-ctr`, `aes192-ctr`, `aes256-ctr`. Please note that the ciphers disabled by default are insecure, you should expect that an active attacker can recover plaintext if you enable them.
- `macs`, list of strings. Available MAC (message authentication code) algorithms in preference order. Leave empty to use default values. The supported values are: `hmac-sha2-256-etm@openssh.com`, `hmac-sha2-256`, `hmac-sha2-512-etm@openssh.com`, `hmac-sha2-512`, `hmac-sha1`, `hmac-sha1-96`. Default values: `hmac-sha2-256-etm@openssh.com`, `hmac-sha2-256`.
- `public_key_algorithms`, list of strings. Public key algorithms that the server will accept for client authentication. The supported values are: `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `rsa-sha2-512`, `rsa-sha2-256`, `ssh-rsa`, `ssh-dss`, `ssh-ed25519`, `sk-ssh-ed25519@openssh.com`, `sk-ecdsa-sha2-nistp256@openssh.com`. Default values: `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, `rsa-sha2-512`, `rsa-sha2-256`, `ssh-ed25519`, `sk-ssh-ed25519@openssh.com`, `sk-ecdsa-sha2-nistp256@openssh.com`.
- `trusted_user_ca_keys`, list of public keys paths of certificate authorities that are trusted to sign user certificates for authentication. The paths can be absolute or relative to the configuration directory.
- `revoked_user_certs_file`, path to a file containing the revoked user certificates. The path can be absolute or relative to the configuration directory. It must contain a JSON list with the public key fingerprints of the revoked certificates. Example content: `["SHA256:bsBRHC/xgiqBJdSuvSTNpJNLTISP/G356jNMCRYC5Es","SHA256:119+8cL/HH+NLMawRsJx6CzPF1I3xC+jpM60bQHXGE8"]`. The revocation list can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows. Default: "".
- `login_banner_file`, path to the login banner file. The contents of the specified file, if any, are sent to the remote user before authentication is allowed. It can be a path relative to the config dir or an absolute one. Leave empty to disable login banner.
- `enabled_ssh_commands`, list of enabled SSH commands. `*` enables all supported commands. More information can be found [here](ssh.md#ssh-commands).
- `keyboard_interactive_authentication`, boolean. This setting specifies whether keyboard interactive authentication is allowed. If no keyboard interactive hook or auth plugin is defined the default is to prompt for the user password and then the one time authentication code, if defined. Default: `true`.
- `keyboard_interactive_auth_hook`, string. Absolute path to an external program or an HTTP URL to invoke for keyboard interactive authentication. See [Keyboard Interactive Authentication](keyboard-interactive.md) for more details.
- `password_authentication`, boolean. Set to false to disable password authentication. This setting will disable multi-step authentication method using public key + password too. It is useful for public key only configurations if you need to manage old clients that will not attempt to authenticate with public keys if the password login method is advertised. Default: `true`.

## FTP server

Supported configuration parameters for the `ftpd` section:

- `bindings`, list of structs. Each struct has the following fields:
  - `port`, integer. The port used for serving FTP requests. 0 means disabled. Default: 0.
  - `address`, string. Leave blank to listen on all available network interfaces. Default: "".
  - `apply_proxy_config`, boolean. If enabled the common proxy configuration, if any, will be applied. Please note that we expect the proxy header on control and data connections. Default `true`.
  - `tls_mode`, integer. 0 means accept both cleartext and encrypted sessions. 1 means TLS is required for both control and data connection. 2 means implicit TLS.Please check that a proper TLS config is in place if you set `tls_mode` is different from 0.
  - `tls_session_reuse`, integer. 0 means session reuse is not checked, clients may or may not reuse TLS sessions. 1 means TLS session reuse is required for explicit FTPS. Legacy reuse method based on session IDs is not supported, clients must use session tickets. Session reuse is not supported for implicit TLS. Default: `0`.
  - `certificate_file`, string. Binding specific TLS certificate. This can be an absolute path or a path relative to the config dir.
  - `certificate_key_file`, string. Binding specific private key matching the above certificate. This can be an absolute path or a path relative to the config dir. If not set the global ones will be used, if any.
  - `min_tls_version`, integer. Defines the minimum version of TLS to be enabled. `12` means TLS 1.2 (and therefore TLS 1.2 and TLS 1.3 will be enabled),`13` means TLS 1.3, `10` means TLS 1.0, `11` means TLS 1.1. Default: `12`.
  - `force_passive_ip`, ip address. External IP address for passive connections. Leave empty to autodetect. If not empty, it must be a valid IPv4 address. Default: "".
  - `passive_ip_overrides`, list of struct that allows to return a different passive ip based on the client IP address. Each struct has the following fields:
    - `networks`, list of strings. Each string must define a network in CIDR notation, for example 192.168.1.0/24.
    - `ip`, string. Passive IP to return if the client IP address belongs to the defined networks. Empty means autodetect.
  - `passive_host`, string. Hostname for passive connections. This hostname will be resolved each time a passive connection is requested and this can, depending on the DNS configuration, take a noticeable amount of time. Enable this setting only if you have a dynamic IP address. Default: "".
  - `client_auth_type`, integer. Set to `1` to require a client certificate and verify it. Set to `2` to request a client certificate during the TLS handshake and verify it if given, in this mode the client is allowed not to send a certificate. At least one certification authority must be defined in order to verify client certificates. If no certification authority is defined, this setting is ignored. Default: 0.
  - `tls_cipher_suites`, list of strings. List of supported cipher suites for TLS version 1.2 and below. If empty, a default list of secure cipher suites is used, with a preference order based on hardware performance. Note that TLS 1.3 ciphersuites are not configurable. The supported ciphersuites names are defined [here](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Any invalid name will be silently ignored. The order matters, the ciphers listed first will be the preferred ones. Default: empty.
  - `passive_connections_security`, integer. Defines the security checks for passive data connections. Set to `0` to require matching peer IP addresses of control and data connection. Set to `1` to disable any checks. Please note that if you run the FTP service behind a proxy you must enable the proxy protocol for control and data connections. Default: `0`.
  - `active_connections_security`, integer. Defines the security checks for active data connections. The supported values are the same as described for `passive_connections_security`. Please note that disabling the security checks you will make the FTP service vulnerable to bounce attacks on active data connections, so change the default value only if you are on a trusted/internal network. Default: `0`.
  - `ignore_ascii_transfer_type`, integer. Set to `1` to silently ignore any client requests to perform ASCII translations via the `TYPE` command. useful in circumstances involving older/mainframe clients and EBCDIC files. This behavior can be Default: `0`.
  - `debug`, boolean. If enabled any FTP command will be logged. This will generate a lot of logs. Enable only if you are investigating a client compatibility issue or something similar. You shouldn't leave this setting enabled for production servers. Default `false`.
- `banner_file`, path to the banner file. The contents of the specified file, if any, are displayed when someone connects to the server. It can be a path relative to the config dir or an absolute one. Leave empty to disable.
- `active_transfers_port_non_20`, boolean. Do not impose the port 20 for active data transfers. Enabling this option allows to run SFTPGo with less privilege. Default: `true`.
- `passive_port_range`, struct containing the key `start` and `end`. Port Range for data connections. Random if not specified. Default range is 50000-50100.
- `disable_active_mode`, boolean. Set to `true` to disable active FTP, default `false`.
- `enable_site`, boolean. Set to true to enable the FTP SITE command. We support `chmod` and `symlink` if SITE support is enabled. Default `false`
- `hash_support`, integer. Set to `1` to enable FTP commands that allow to calculate the hash value of files. These FTP commands will be enabled: `HASH`, `XCRC`, `MD5/XMD5`, `XSHA/XSHA1`, `XSHA256`, `XSHA512`. Please keep in mind that to calculate the hash we need to read the whole file, for remote backends this means downloading the file, for the encrypted backend this means decrypting the file. Default `0`.
- `combine_support`, integer. Set to 1 to enable support for the non standard `COMB` FTP command. Combine is only supported for local filesystem, for cloud backends it has no advantage as it will download the partial files and will upload the combined one. Cloud backends natively support multipart uploads. Default `0`.
- `certificate_file`, string. Certificate for FTPS. This can be an absolute path or a path relative to the config dir.
- `certificate_key_file`, string. Private key matching the above certificate. This can be an absolute path or a path relative to the config dir. A certificate and the private key are required to enable explicit and implicit TLS. Certificate and key files can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows. The certificates are also polled for changes every 8 hours.
- `ca_certificates`, list of strings. Set of root certificate authorities to be used to verify client certificates.
- `ca_revocation_lists`, list of strings. Set a revocation lists, one for each root CA, to be used to check if a client certificate has been revoked. The revocation lists can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows.

## WebDAV Server

Supported configuration parameters for the `webdavd` section:

- `bindings`, list of structs. Each struct has the following fields:
  - `port`, integer. The port used for serving WebDAV requests. 0 means disabled. Default: 0.
  - `address`, string. Leave blank to listen on all available network interfaces. Default: "".
  - `enable_https`, boolean. Set to `true` and provide both a certificate and a key file to enable HTTPS connection for this binding. Default `false`.
  - `certificate_file`, string. Binding specific TLS certificate. This can be an absolute path or a path relative to the config dir.
  - `certificate_key_file`, string. Binding specific private key matching the above certificate. This can be an absolute path or a path relative to the config dir. If not set the global ones will be used, if any.
  - `min_tls_version`, integer. Defines the minimum version of TLS to be enabled. `12` means TLS 1.2 (and therefore TLS 1.2 and TLS 1.3 will be enabled),`13` means TLS 1.3, `10` means TLS 1.0, `11` means TLS 1.1. Default: `12`.
  - `client_auth_type`, integer. Set to `1` to require a client certificate and verify it. Set to `2` to request a client certificate during the TLS handshake and verify it if given, in this mode the client is allowed not to send a certificate. At least one certification authority must be defined in order to verify client certificates. If no certification authority is defined, this setting is ignored. Default: 0.
  - `tls_cipher_suites`, list of strings. List of supported cipher suites for TLS version 1.2 and below. If empty, a default list of secure cipher suites is used, with a preference order based on hardware performance. Note that TLS 1.3 ciphersuites are not configurable. The supported ciphersuites names are defined [here](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Any invalid name will be silently ignored. The order matters, the ciphers listed first will be the preferred ones. Default: empty.
  - `tls_protocols`, list of string. HTTPS protocols in preference order. Supported values: `http/1.1`, `h2`. Default: `http/1.1`, `h2`.
  - `prefix`, string. Prefix for WebDAV resources, if empty WebDAV resources will be available at the `/` URI. If defined it must be an absolute URI, for example `/dav`. Default: "".
  - `proxy_mode`, integer. Set to `1` to use the proxy protocol configuration defined within the `common` section instead of the proxy header configuration. Default: `0`.
  - `proxy_allowed`, list of IP addresses and IP ranges allowed to set client IP proxy header such as `X-Forwarded-For`. Any client IP proxy headers, if set on requests from a connection address not in this list, will be silently ignored. Default: empty.
  - `client_ip_proxy_header`, string. Defines the allowed client IP proxy header such as `X-Forwarded-For`, `X-Real-IP` etc. Default: empty
  - `client_ip_header_depth`, integer. Some client IP headers such as `X-Forwarded-For` can contain multiple IP address, this setting define the position to trust starting from the right. For example if we have: `10.0.0.1,11.0.0.1,12.0.0.1,13.0.0.1` and the depth is `0`, SFTPGo will use `13.0.0.1` as client IP, if depth is `1`, `12.0.0.1` will be used and so on. Default: `0`.
  - `disable_www_auth_header`, boolean. Set to `true` to not add the WWW-Authenticate header after an authentication failure, only the `401` status code will be sent. Default: `false`.
- `certificate_file`, string. Certificate for WebDAV over HTTPS. This can be an absolute path or a path relative to the config dir.
- `certificate_key_file`, string. Private key matching the above certificate. This can be an absolute path or a path relative to the config dir. A certificate and a private key are required to enable HTTPS connections. Certificate and key files can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows.
- `ca_certificates`, list of strings. Set of root certificate authorities to be used to verify client certificates.
- `ca_revocation_lists`, list of strings. Set a revocation lists, one for each root CA, to be used to check if a client certificate has been revoked. The revocation lists can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows. The certificates are also polled for changes every 8 hours.
- `cors` struct containing CORS configuration. SFTPGo uses [Go CORS handler](https://github.com/rs/cors){:target="_blank"}, please refer to upstream documentation for fields meaning and their default values.
  - `enabled`, boolean, set to true to enable CORS.
  - `allowed_origins`, list of strings.
  - `allowed_methods`, list of strings.
  - `allowed_headers`, list of strings.
  - `exposed_headers`, list of strings.
  - `allow_credentials` boolean.
  - `max_age`, integer.
  - `options_passthrough`, boolean.
  - `options_success_status`, integer.
  - `allow_private_network`, boolean.
- `cache` struct containing cache configurations.
  - `users`, cache configuration for the authenticated users.
    - `expiration_time`, integer. Expiration time, in minutes, for the cached users. 0 means unlimited. Default: 0.
    - `max_size`, integer. Maximum number of users to cache. 0 means unlimited. Default: 50.
  - `mime_types`, cache configuration for mime types.
    - `enabled`, boolean, set to true to enable mime types caching. Default: `true`.
    - `max_size`, integer. Maximum number of mime types to cache. 0 means no cache. Default: 1000.
    - `custom_mappings`, additional mime types mapping. This is a platform independet way to add few additional mappings. You can set a limited number of mappings here, if you want to add a large list use the method provided by the OS of your choice. List of struct, each struct has the following fields:
      - `ext`, string, file extension including the dot, for example `.json`
      - `mime`, string, mime type, for example `application/json`

## Data provider

Supported configuration parameters for the `data_provider` section:

- `driver`, string. Supported drivers are `sqlite`, `mysql`, `postgresql`, `cockroachdb`, `bolt`, `memory`
- `name`, string. Database name. For driver `sqlite` this can be the database name relative to the config dir or the absolute path to the SQLite database. For driver `memory` this is the (optional) path relative to the config dir or the absolute path to the provider dump, obtained using the `dumpdata` REST API, to load. This dump will be loaded at startup and can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows. The `memory` provider will not modify the provided file so quota usage and last login will not be persisted. If you plan to use a SQLite database over a `cifs` network share (this is not recommended in general) you must use the `nobrl` mount option otherwise you will get the `database is locked` error. Some users reported that the `bolt` provider works fine over `cifs` shares.
- `host`, string. Database host. For `postgresql` and `cockroachdb` drivers you can specify multiple hosts separated by commas. Leave empty for drivers `sqlite`, `bolt` and `memory`
- `port`, integer. Database port. Leave empty for drivers `sqlite`, `bolt` and `memory`
- `username`, string. Database user. Leave empty for drivers `sqlite`, `bolt` and `memory`
- `password`, string. Database password. Leave empty for drivers `sqlite`, `bolt` and `memory`
- `sslmode`, integer. Used for drivers `mysql` and `postgresql`. 0 disable TLS connections, 1 require TLS, 2 set TLS mode to `verify-ca` for driver `postgresql` and `skip-verify` for driver `mysql`, 3 set TLS mode to `verify-full` for driver `postgresql` and `preferred` for driver `mysql`, 4 set the TLS mode to `prefer` for driver `postgresql`, 5 set the TLS mode to `allow` for driver `postgresql`
- `root_cert`, string. Path to the root certificate authority used to verify that the server certificate was signed by a trusted CA
- `disable_sni`, boolean. Allows to opt out Server Name Indication (SNI) for TLS connections. Default: `false`
- `target_session_attrs`, string. This is a `postgresql` and `cockroachdb` specific option. It determines whether the session must have certain properties to be acceptable. It's typically used in combination with multiple host names to select the first acceptable alternative among several hosts. Supported values: `any`, `read-write`, `read-only`, `primary`, `standby`, `prefer-standby`. If empty, `any` is assumed. If you explicitly set `any` the connections will be randomly distributed among the specified hosts
- `client_cert`, string. Path to the client certificate for two-way TLS authentication
- `client_key`,string. Path to the client key for two-way TLS authentication
- `connection_string`, string. Provide a custom database connection string. If not empty, this connection string will be used instead of building one using the previous parameters. Leave empty for drivers `bolt` and `memory`
- `sql_tables_prefix`, string. Prefix for SQL tables
- `track_quota`, integer. Set the preferred mode to track users quota between the following choices:
  - 0, disable quota tracking. REST API to scan users home directories/virtual folders and update quota will do nothing
  - 1, quota is updated each time a user uploads or deletes a file, even if the user has no quota restrictions
  - 2, quota is updated each time a user uploads or deletes a file, but only for users with quota restrictions and for virtual folders. With this configuration, the `quota scan` and `folder_quota_scan` REST API can still be used to periodically update space usage for users without quota restrictions and for folders
- `delayed_quota_update`, integer. This configuration parameter defines the number of seconds to accumulate quota updates. If there are a lot of close uploads, accumulating quota updates can save you many queries to the data provider. If you want to track quotas, a scheduled quota update is recommended in any case, the stored quota may be incorrect for several reasons, such as an unexpected shutdown while uploading files, temporary provider failures, files copied outside of SFTPGo, and so on. You can use the [eventmanager](eventmanager.md) to schedule a periodic quota update. 0 means immediate quota update.
- `pool_size`, integer. Sets the maximum number of open connections for `mysql` and `postgresql` driver. Default 0 (unlimited)
- `users_base_dir`, string. Users default base directory. If no home dir is defined while adding a new user, and this value is a valid absolute path, then the user home dir will be automatically defined as the path obtained joining the base dir and the username
- `actions`, struct. It contains the command to execute and/or the HTTP URL to notify and the trigger conditions. See [Custom Actions](custom-actions.md) for more details
  - `execute_on`, list of strings. Valid values are `add`, `update`, `delete`. `update` action will not be fired for internal updates such as the last login or the user quota fields.
  - `execute_for`, list of strings. Defines the provider objects that trigger the action. Valid values are `user`, `folder`, `group`, `admin`, `api_key`, `share`, `event_action`, `event_rule`.
  - `hook`, string. Absolute path to the command to execute or HTTP URL to notify.
- `external_auth_hook`, string. Absolute path to an external program or an HTTP URL to invoke for users authentication. See [External Authentication](external-auth.md) for more details. Leave empty to disable.
- `external_auth_scope`, integer. 0 means all supported authentication scopes (passwords, public keys and keyboard interactive). 1 means passwords only. 2 means public keys only. 4 means key keyboard interactive only. 8 means TLS certificate. The flags can be combined, for example 6 means public keys and keyboard interactive
- `credentials_path`, string. It defines the directory for storing user provided credential files such as Google Cloud Storage credentials. This can be an absolute path or a path relative to the config dir
- `pre_login_hook`, string. Absolute path to an external program or an HTTP URL to invoke to modify user details just before the login. See [Dynamic user modification](dynamic-user-mod.md) for more details. Leave empty to disable.
- `post_login_hook`, string. Absolute path to an external program or an HTTP URL to invoke to notify a successful or failed login. See [Post-login hook](post-login-hook.md) for more details. Leave empty to disable.
- `post_login_scope`, defines the scope for the post-login hook. 0 means notify both failed and successful logins. 1 means notify failed logins. 2 means notify successful logins.
- `check_password_hook`, string.  Absolute path to an external program or an HTTP URL to invoke to check the user provided password. See [Check password hook](check-password-hook.md) for more details. Leave empty to disable.
- `check_password_scope`, defines the scope for the check password hook. 0 means all protocols, 1 means SSH, 2 means FTP, 4 means WebDAV. You can combine the scopes, for example 6 means FTP and WebDAV.
- `password_hashing`, struct. It contains the configuration parameters to be used to generate the password hash. SFTPGo can verify passwords in several formats and uses, by default, the `bcrypt` algorithm to hash passwords in plain-text before storing them inside the data provider. These options allow you to customize how the hash is generated.
  - `argon2_options`, struct containing the options for argon2id hashing algorithm. The `memory` and `iterations` parameters control the computational cost of hashing the password. The higher these figures are, the greater the cost of generating the hash and the longer the runtime. It also follows that the greater the cost will be for any attacker trying to guess the password. If the code is running on a machine with multiple cores, then you can decrease the runtime without reducing the cost by increasing the `parallelism` parameter. This controls the number of threads that the work is spread across.
    - `memory`, unsigned integer. The amount of memory used by the algorithm (in kibibytes). Default: 65536.
    - `iterations`, unsigned integer. The number of iterations over the memory. Default: 1.
    - `parallelism`. unsigned 8 bit integer. The number of threads (or lanes) used by the algorithm. Default: 2.
  - `bcrypt_options`, struct containing the options for bcrypt hashing algorithm
    - `cost`, integer between 4 and 31. Default: 10
  - `algo`, string. Algorithm to use for hashing passwords. Available algorithms: `argon2id`, `bcrypt`. For bcrypt hashing we use the `$2a$` prefix. Default: `bcrypt`
- `password_validation` struct. It defines the password validation rules for admins and protocol users.
  - `admins`, struct. It defines the password validation rules for SFTPGo admins.
    - `min_entropy`, float. Defines the minimum password entropy. Take a look [here](https://github.com/wagslane/go-password-validator#what-entropy-value-should-i-use){:target="_blank"} for more details. `0` means disabled, any password will be accepted. Default: `0`.
  - `users`, struct. It defines the password validation rules for SFTPGo protocol users.
    - `min_entropy`, float. This value is used as fallback if no more specific password strength is set at user/group level. Default: `0`.
- `password_caching`, boolean. Verifying argon2id passwords has a high memory and computational cost, verifying bcrypt passwords has a high computational cost, by enabling, in memory, password caching you reduce these costs. Default: `true`
- `update_mode`, integer. Defines how the database will be initialized/updated. 0 means automatically. 1 means manually using the initprovider sub-command.
- `create_default_admin`, boolean. Before you can use SFTPGo you need to create an admin account. If you open the admin web UI, a setup screen will guide you in creating the first admin account. You can automatically create the first admin account by enabling this setting and setting the environment variables `SFTPGO_DEFAULT_ADMIN_USERNAME` and `SFTPGO_DEFAULT_ADMIN_PASSWORD`. You can also create the first admin by loading initial data. This setting has no effect if an admin account is already found within the data provider. Default `false`.
- `naming_rules`, integer. Naming rules for usernames, folder, group, role and object names in general. `0` means no rules. `1` means you can use any UTF-8 character. The names are used in URIs for REST API and Web admin. If not set only unreserved URI characters are allowed: ALPHA / DIGIT / "-" / "." / "_" / "~". `2` means names are converted to lowercase before saving/matching and so case insensitive matching is possible. `4` means trimming trailing and leading white spaces before saving/matching, the WebAdmin needs this setting to work properly. Rules can be combined, for example `3` means both converting to lowercase and allowing any UTF-8 character. Enabling these options for existing installations could be backward incompatible, some users could be unable to login, for example existing users with mixed cases in their usernames. You have to ensure that all existing users respect the defined rules. Default: `5`.
- `is_shared`, integer. If the data provider is shared across multiple SFTPGo instances, set this parameter to `1`. `MySQL`, `PostgreSQL` and `CockroachDB` can be shared, this setting is ignored for other data providers. For shared data providers, active transfers are persisted in the database and thus quota checks between ongoing transfers will work cross multiple instances. Password reset requests and OIDC tokens/states are also persisted in the database if the provider is shared. For shared data providers, scheduled event actions are only executed on a single SFTPGo instance by default, you can override this behavior on a per-action basis. The database table `shared_sessions` is used only to store temporary sessions. In performance critical installations, you might consider using a database-specific optimization, for example you might use an `UNLOGGED` table for PostgreSQL. This optimization in only required in very limited use cases. Default: `0`.
- `node`, struct. Node-specific configurations to allow inter-node communications. If your provider is shared across multiple nodes, the nodes can exchange information to present a uniform view for node-specific data. The current implementation allows to obtain active connections from all nodes. Nodes connect to each other using the REST API.
  - `host`, string. IP address or hostname that other nodes can use to connect to this node via REST API. Empty means inter-node communications disabled. Default: empty.
  - `port`, integer. The port that other nodes can use to connect to this node via REST API. Default: `0`
  - `proto`, string. Supported values `http` or `https`. For `https` the configurations for http clients is used, so you can, for example, enable mutual TLS authentication. Default: `http`
- `backups_path`, string. Path to the backup directory. This can be an absolute path or a path relative to the config dir. We don't allow backups in arbitrary paths for security reasons.

## HTTP server

Supported configuration parameters for the `httpd` section (REST API, WebAdmin, WebClient):

- `bindings`, list of structs. Each struct has the following fields:
  - `port`, integer. The port used for serving HTTP requests. Default: 8080.
  - `address`, string. Leave blank to listen on all available network interfaces. On *NIX you can specify an absolute path to listen on a Unix-domain socket Default: blank.
  - `enable_web_admin`, boolean. Set to `false` to disable the built-in web admin for this binding. You also need to define `templates_path` and `static_files_path` to use the built-in web admin interface. Default `true`.
  - `enable_web_client`, boolean. Set to `false` to disable the built-in web client for this binding. You also need to define `templates_path` and `static_files_path` to use the built-in web client interface. Default `true`.
  - `enable_rest_api`, boolean. Set to `false` to disable REST API. Default `true`.
  - `enabled_login_methods`, integer. Defines the login methods available for the WebAdmin and WebClient UIs. `0` means any configured method: username/password login form and OIDC, if enabled. `1` means OIDC for the WebAdmin UI. `2` means OIDC for the WebClient UI. `4` means login form for the WebAdmin UI. `8` means login form for the WebClient UI. You can combine the values. For example `3` means that you can only login using OIDC on both WebClient and WebAdmin UI. DEPRECATED: use `disabled_login_methods`. Default: `0`.
  - `disabled_login_methods`, integer. Defines the disabled login methods. `0` means all available login methods are enabled. `1` means OIDC for the WebAdmin UI. `2` means OIDC for the WebClient UI. `4` means login form for the WebAdmin UI. `8` means login form for the WebClient UI. You can combine the values. `16` means the admin token endpoint for REST API. `32` means the user token endpoint for REST API. `64` means admin API key login. `128` means user API key login. For example `252` means that you can only login using OIDC on both WebClient and WebAdmin UI. Default: `0`
  - `enable_https`, boolean. Set to `true` and provide both a certificate and a key file to enable HTTPS connection for this binding. Default `false`.
  - `certificate_file`, string. Binding specific TLS certificate. This can be an absolute path or a path relative to the config dir.
  - `certificate_key_file`, string. Binding specific private key matching the above certificate. This can be an absolute path or a path relative to the config dir. If not set the global ones will be used, if any.
  - `min_tls_version`, integer. Defines the minimum version of TLS to be enabled. `12` means TLS 1.2 (and therefore TLS 1.2 and TLS 1.3 will be enabled),`13` means TLS 1.3, `10` means TLS 1.0, `11` means TLS 1.1. Default: `12`.
  - `client_auth_type`, integer. Set to `1` to require client certificate authentication in addition to JWT/Web authentication. You need to define at least a certificate authority for this to work. Default: 0.
  - `tls_cipher_suites`, list of strings. List of supported cipher suites for TLS version 1.2 and below. If empty, a default list of secure cipher suites is used, with a preference order based on hardware performance. Note that TLS 1.3 ciphersuites are not configurable. The supported ciphersuites names are defined [here](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Any invalid name will be silently ignored. The order matters, the ciphers listed first will be the preferred ones. Default: empty.
  - `tls_protocols`, list of string. HTTPS protocols in preference order. Supported values: `http/1.1`, `h2`. Default: `http/1.1`, `h2`.
  - `proxy_mode`, integer. Set to `1` to use the proxy protocol configuration defined within the `common` section instead of the proxy header configuration. Default: `0`.
  - `proxy_allowed`, list of IP addresses and IP ranges allowed to set client IP proxy header such as `X-Forwarded-For`, `X-Real-IP` and any other headers defined in the `security` section. Any of the indicated headers, if set on requests from a connection address not in this list, will be silently ignored. Default: empty.
  - `client_ip_proxy_header`, string. Defines the allowed client IP proxy header such as `X-Forwarded-For`, `X-Real-IP` etc. Default: empty
  - `client_ip_header_depth`, integer. Some client IP headers such as `X-Forwarded-For` can contain multiple IP address, this setting define the position to trust starting from the right. For example if we have: `10.0.0.1,11.0.0.1,12.0.0.1,13.0.0.1` and the depth is `0`, SFTPGo will use `13.0.0.1` as client IP, if depth is `1`, `12.0.0.1` will be used and so on. Default: `0`.
  - `hide_login_url`, integer. If both web admin and web client are enabled each login page will show a link to the other one. This setting allows to hide this link. 0 means that the login links are displayed on both admin and client login page. This is the default. 1 means that the login link to the web client login page is hidden on admin login page. 2 means that the login link to the web admin login page is hidden on client login page. The flags can be combined, for example 3 will disable both login links.
  - `render_openapi`, boolean. Set to `false` to disable serving of the OpenAPI schema and renderer. Default `true`.
  - `languages`, list of strings. Supporrted values: `en`, `it`, `de`, `fr`. Default: `en`.
  - `oidc`, struct. Defines the OpenID connect configuration. OpenID integration allows you to map your identity provider users to SFTPGo users and so you can login to SFTPGo Web Client and Web Admin user interfaces using your identity provider. The following fields are supported:
    - `config_url`, string. Identifier for the service. If defined, SFTPGo will add `/.well-known/openid-configuration` to this url and attempt to retrieve the provider configuration on startup. SFTPGo will refuse to start if it fails to connect to the specified URL. Default: blank.
    - `client_id`, string. Defines the application's ID. Default: blank.
    - `client_secret`, string. Defines the application's secret. Default: blank.
    - `client_secret_file`, string. Defines the path to a file containing the application secret. This can be an absolute path or a path relative to the config dir. If not empty, it takes precedence over `client_secret`. Default: blank.
    - `redirect_base_url`, string. Defines the base URL to redirect to after OpenID authentication. The suffix `/web/oidc/redirect` will be added to this base URL, adding also the `web_root` if configured. Default: blank.
    - `username_field`, string. Defines the ID token claims field to map to the SFTPGo username. Default: blank.
    - `scopes`, list of strings. Request the OAuth provider to provide the scope information from an authenticated users. The `openid` scope is mandatory. Default: `"openid", "profile", "email"`.
    - `role_field`, string. Defines the optional ID token claims field to map to a SFTPGo role. If the defined ID token claims field is set to `admin` the authenticated user is mapped to an SFTPGo admin. You don't need to specify this field if you want to use OpenID only for the Web Client UI. If the field is inside a nested structure, you can use the dot notation to traverse the structures. Default: blank.
    - `implicit_roles`, boolean. If set, the `role_field` is ignored and the SFTPGo role is assumed based on the login link used. Default: `false`.
    - `custom_fields`, list of strings. Custom token claims fields to pass to the pre-login hook. Default: empty.
    - `insecure_skip_signature_check`, boolean. This setting causes SFTPGo to skip JWT signature validation. It's intended for special cases where providers, such as Azure, use the `none` algorithm. Skipping the signature validation can cause security issues. Default: `false`.
    - `debug`, boolean. If set, the received id tokens will be logged at debug level. Default: `false`.
  - `security`, struct. Defines security headers to add to HTTP responses and allows to restrict allowed hosts. The following parameters are supported:
    - `enabled`, boolean. Set to `true` to enable security configurations. Default: `false`.
    - `allowed_hosts`, list of strings. Fully qualified domain names that are allowed. An empty list allows any and all host names. Default: empty.
    - `allowed_hosts_are_regex`, boolean. Determines if the provided allowed hosts contains valid regular expressions. Default: `false`.
    - `hosts_proxy_headers`, list of string. Defines a set of header keys that may hold a proxied hostname value for the request, for example `X-Forwarded-Host`. Default: empty.
    - `https_redirect`, boolean. Set to `true` to redirect HTTP requests to HTTPS. If you redirect from port `80` and you get your TLS certificates using the built-in ACME protocol and the `HTTP-01` challenge type, you need to use the webroot method and set the ACME web root to a path writable by SFTPGo in order to renew your certificates. Default: `false`.
    - `https_host`, string. Defines the host name that is used to redirect HTTP requests to HTTPS. Default is blank, which indicates to use the same host. For example, if `https_redirect` is enabled and `https_host` is blank, a request for `http://127.0.0.1/web/client/login` will be redirected to `https://127.0.0.1/web/client/login`, if `https_host` is set to `www.example.com` the same request will be redirected to `https://www.example.com/web/client/login`.
    - `https_proxy_headers`, list of struct, each struct contains the fields `key` and `value`. Defines a a list of header keys with associated values that would indicate a valid https request. For example `key` could be `X-Forwarded-Proto` and `value` `https`. Default: empty.
    - `sts_seconds`, integer. Defines the max-age of the `Strict-Transport-Security` header. This header will be included for `https` responses or for HTTP request if the request includes a defined HTTPS proxy header. Default: `0`, which would NOT include the header.
    - `sts_include_subdomains`, boolean. Set to `true`, the `includeSubdomains` will be appended to the `Strict-Transport-Security` header. Default: `false`.
    - `sts_preload`, boolean. Set to true, the `preload` flag will be appended to the `Strict-Transport-Security` header. Default: `false`.
    - `content_type_nosniff`, boolean. Set to `true` to add the `X-Content-Type-Options` header with the value `nosniff`. Default: `false`.
    - `content_security_policy`, string. Allows to set the `Content-Security-Policy` header value. Default: blank.
    - `permissions_policy`, string. Allows to set the `Permissions-Policy` header value. Default: blank.
    - `cross_origin_opener_policy`, string. Allows to set the `Cross-Origin-Opener-Policy` header value. Default: blank.
    - `cross_origin_resource_policy`, string. Allows to set the `Cross-Origin-Resource-Policy` header value. Default: blank.
    - `cross_origin_embedder_policy`, string. Allows to set the `Cross-Origin-Embedder-Policy` header value. Default: blank.
    - `cache_control`, string. Allows to set the `Cache-Control` header. Set to `private` to disable caching for dynamic pages. Default: blank.
  - `branding`, struct. Defines the supported customizations to suit your brand. It contains the `web_admin` and `web_client` structs that define customizations for the WebAdmin and the WebClient UIs. Each customization struct contains the following fields:
    - `name`, string. Defines the UI name
    - `short_name`, string. Defines the short name to show next to the logo image and on the login page
    - `favicon_path`, string. Path to the favicon relative to `static_files_path`. For example, if you create a directory named `branding` inside the static dir and put the `favicon.png` file in it, you must set `/branding/favicon.png` as path.
    - `logo_path`, string. Path to your logo relative to `static_files_path`. The preferred image size is 256x256 pixel
    - `disclaimer_name`, string. Name for your optional disclaimer
    - `disclaimer_path`, string. Path to the HTML page with the disclaimer relative to `static_files_path` or an absolute URL (http or https).
    - `default_css`, list of strings. Optional path to custom CSS files, relative to `static_files_path`, which replaces the default CSS
    - `extra_css`, list of strings. Defines the paths, relative to `static_files_path`, to additional CSS files
- `templates_path`, string. Path to the HTML web templates. This can be an absolute path or a path relative to the config dir
- `static_files_path`, string. Path to the static files for the web interface. This can be an absolute path or a path relative to the config dir. If both `templates_path` and `static_files_path` are empty the built-in web interface will be disabled
- `openapi_path`, string. Path to the directory that contains the OpenAPI schema and the default renderer. This can be an absolute path or a path relative to the config dir. If empty the OpenAPI schema and the renderer will not be served regardless of the `render_openapi` directive
- `web_root`, string.  Defines a base URL for the web admin and client interfaces. If empty web admin and client resources will be available at the root ("/") URI. If defined it must be an absolute URI or it will be ignored
- `certificate_file`, string. Certificate for HTTPS. This can be an absolute path or a path relative to the config dir.
- `certificate_key_file`, string. Private key matching the above certificate. This can be an absolute path or a path relative to the config dir. If both the certificate and the private key are provided, you can enable HTTPS for the configured bindings. Certificate and key files can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows. The certificates are also polled for changes every 8 hours.
- `ca_certificates`, list of strings. Set of root certificate authorities to be used to verify client certificates.
- `ca_revocation_lists`, list of strings. Set a revocation lists, one for each root CA, to be used to check if a client certificate has been revoked. The revocation lists can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows.
- `signing_passphrase`, string. Passphrase to use to derive the signing key for JWT and CSRF tokens. If empty a random signing key will be generated each time SFTPGo starts. If you set a signing passphrase you should consider rotating it periodically for added security.
- `signing_passphrase_file`, string. Defines the path to a file containing the signing passphrase. This can be an absolute path or a path relative to the config dir. If not empty, it takes precedence over `signing_passphrase`. Default: blank.
- `token_validation`, integer. Defines how to validate JWT tokens, cookies and CSRF tokens. By default a token must be used by the same IP for which it was issued, set to `1` to disable this requirement. Set to `2` to invalidate admin and user tokens issued before the last update. Flags can be combined. Default: `0`.
- `cookie_lifetime`, integer. Defines the duration in minutes for WebAdmin and WebClient cookies. Cookies are automatically refreshed if there is user activity and the maximum duration is 12 hours. An invalid value is silently ignored. Maximum allowed value `720`, default: `20`.
- `share_cookie_lifetime`, integer. Defines the duration in minutes for public sharing cookies. An invalid value is silently ignored. Maximum allowed value `720`, default: `120`.
- `jwt_lifetime`, integer. Defines the duration in minutes for REST API tokens. An invalid value is silently ignored. Maximum allowed value `720`, default: `20`.
- `max_upload_file_size`, integer. Defines the maximum request body size, in bytes, for Web Client/API HTTP upload requests. `0` means no limit. Default: `0`.
- `cors` struct containing CORS configuration. SFTPGo uses [Go CORS handler](https://github.com/rs/cors){:target="_blank"}, please refer to upstream documentation for fields meaning and their default values.
  - `enabled`, boolean, set to `true` to enable CORS.
  - `allowed_origins`, list of strings.
  - `allowed_methods`, list of strings.
  - `allowed_headers`, list of strings.
  - `exposed_headers`, list of strings.
  - `allow_credentials` boolean.
  - `max_age`, integer.
  - `options_passthrough`, boolean.
  - `options_success_status`, integer.
  - `allow_private_network`, boolean.
- `setup` struct containing configurations for the initial setup screen
  - `installation_code`, string. If set, this installation code will be required when creating the first admin account. Please note that even if set using an environment variable this field is read at SFTPGo startup and not at runtime. This is not a license key or similar, the purpose here is to prevent anyone who can access to the initial setup screen from creating an admin user. Default: blank.
  - `installation_code_hint`, string. Description for the installation code input field. Default: `Installation code`.
- `hide_support_link`, boolean. If set, the link to the [sponsors section](https://github.com/drakkan/sftpgo?tab=readme-ov-file#sponsors) will not appear on the setup screen page. Default: `false`.

## Telemetry

Supported configuration parameters for the `telemetry` section:

- `bind_port`, integer. The port used for serving HTTP requests. Set to 0 to disable HTTP server. Default: 0
- `bind_address`, string. Leave blank to listen on all available network interfaces. On \*NIX you can specify an absolute path to listen on a Unix-domain socket. Default: `127.0.0.1`
- `enable_profiler`, boolean. Enable the built-in profiler. Default `false`
- `auth_user_file`, string. Path to a file used to store usernames and passwords for basic authentication. This can be an absolute path or a path relative to the config dir. We support HTTP basic authentication, and the file format must conform to the one generated using the Apache `htpasswd` tool. The supported password formats are bcrypt (`$2y$` prefix) and md5 crypt (`$apr1$` prefix). If empty, HTTP authentication is disabled. Authentication will be always disabled for the `/healthz` endpoint.
- `certificate_file`, string. Certificate for HTTPS. This can be an absolute path or a path relative to the config dir.
- `certificate_key_file`, string. Private key matching the above certificate. This can be an absolute path or a path relative to the config dir. If both the certificate and the private key are provided, the server will expect HTTPS connections. Certificate and key files can be reloaded on demand sending a `SIGHUP` signal on Unix based systems and a `paramchange` request to the running service on Windows.
- `min_tls_version`, integer. Defines the minimum version of TLS to be enabled. `12` means TLS 1.2 (and therefore TLS 1.2 and TLS 1.3 will be enabled),`13` means TLS 1.3, `10` means TLS 1.0, `11` means TLS 1.1. Default: `12`.
- `tls_cipher_suites`, list of strings. List of supported cipher suites for TLS version 1.2 and below. If empty, a default list of secure cipher suites is used, with a preference order based on hardware performance. Note that TLS 1.3 ciphersuites are not configurable. The supported ciphersuites names are defined [here](https://github.com/golang/go/blob/master/src/crypto/tls/cipher_suites.go#L53){:target="_blank"}. Any invalid name will be silently ignored. The order matters, the ciphers listed first will be the preferred ones. Default: empty.
- `tls_protocols`, list of string. HTTPS protocols in preference order. Supported values: `http/1.1`, `h2`. Default: `http/1.1`, `h2`.

The telemetry server publishes the following endpoints:

- `/healthz`, health information (for health checks)
- `/metrics`, Prometheus metrics
- `/debug/pprof`, if enabled via the `enable_profiler` configuration key, for profiling, more details [here](profiling.md)

## HTTP clients

HTTP clients are used for executing hooks. Some hooks use a retryable HTTP client, for these hooks you can configure the time between retries and the number of retries. Please check the hook specific documentation to understand which hooks use a retryable HTTP client.

Supported configuration parameters for the `http` section:

- `timeout`, float. Timeout specifies a time limit, in seconds, for requests. For requests with retries this is the timeout for a single request
- `retry_wait_min`, integer. Defines the minimum waiting time between attempts in seconds.
- `retry_wait_max`, integer. Defines the maximum waiting time between attempts in seconds. The backoff algorithm will perform exponential backoff based on the attempt number and limited by the provided minimum and maximum durations.
- `retry_max`, integer. Defines the maximum number of retries if the first request fails.
- `ca_certificates`, list of strings. List of paths to extra CA certificates to trust. The paths can be absolute or relative to the config dir. Adding trusted CA certificates is a convenient way to use self-signed certificates without defeating the purpose of using TLS.
- `certificates`, list of certificate for mutual TLS. Each certificate is a struct with the following fields:
  - `cert`, string. Path to the certificate file. The path can be absolute or relative to the config dir.
  - `key`, string. Path to the key file. The path can be absolute or relative to the config dir.
- `skip_tls_verify`, boolean. if enabled the HTTP client accepts any TLS certificate presented by the server and any host name in that certificate. In this mode, TLS is susceptible to man-in-the-middle attacks. This should be used only for testing.
- `headers`, list of structs. You can define a list of http headers to add to each hook. Each struct has the following fields:
  - `key`, string
  - `value`, string. The header is silently ignored if `key` or `value` are empty
  - `url`, string, optional. If not empty, the header will be added only if the request URL starts with the one specified here

## Commands

External commands are used for executing hooks.
Supported configuration parameters for the `command` section:

- `timeout`, integer. Timeout specifies a time limit, in seconds, to execute external commands. Valid range: `1-300`. Default: `30`
- `env`, list of strings. Environment variables to pass to all the external commands. Global environment variables are cleared, for security reasons, you have to explicitly set any environment variable such as `PATH` etc. if you need them. Each entry is of the form `key=value`. Do not use environment variables prefixed with `SFTPGO_` to avoid conflicts with environment variables that SFTPGo hooks can set. Default: empty
- `commands`, list of structs. Allow to customize configuration per-command. Each struct has the following fields:
  - `path`, string. Define the command path as defined in the hook configuration
  - `timeout`, integer. This value overrides the global timeout if set
  - `env`, list of strings. These values are added to the environment variables defined for all commands, if any. Default: empty
  - `args`, list of strings. Arguments to pass to the command identified by `path`. Default: empty
  - `hook`, string. If not empty this configuration only apply to the specified hook name. Supported hook names: `fs_actions`, `provider_actions`, `startup`, `post_connect`, `post_disconnect`, `data_retention`, `check_password`, `pre_login`, `post_login`, `external_auth`, `keyboard_interactive`. Default: empty

## KMS

Supported configuration parameters for the `kms` section:

- `secrets`
  - `url`, string. Defines the URI to the KMS service. Default: blank.
  - `master_key`, string. Defines the master encryption key as string. Default: blank.
  - `master_key_path`, string. Defines the absolute path to a file containing the master encryption key. If not empty, it takes precedence over `master_key`. Default: blank.

## Multi-factor authentication

Supported configuration parameters for the `mfa` section:

- `totp`, list of struct that define settings for time-based one time passwords (RFC 6238). Each struct has the following fields:
  - `name`, string. Unique configuration name. This name should not be changed if there are users or admins using the configuration. The name is not visible to the authentication apps. Default: `Default`.
  - `issuer`, string. Name of the issuing Organization/Company. Default: `SFTPGo`.
  - `algo`, string. Algorithm to use for HMAC. The supported algorithms are: `sha1`, `sha256`, `sha512`. Currently Google Authenticator app on iPhone seems to only support `sha1`, please check the compatibility with your target apps/device before setting a different algorithm. You can also define multiple configurations, for example one that uses `sha256` or `sha512` and another one that uses `sha1` and instruct your users to use the appropriate configuration for their devices/apps. The algorithm should not be changed if there are users or admins using the configuration. Default: `sha1`.

## SMTP

SMTP configuration enables SFTPGo email sending capabilities.
Supported configuration parameters for the `smtp` section:

- `host`, string. Location of SMTP email server. Leave empty to disable email sending capabilities. Default: blank.
- `port`, integer. Port of SMTP email server.
- `from`, string. From address, for example `SFTPGo <sftpgo@example.com>`. Many SMTP servers reject emails without a `From` header so, if not set, SFTPGo will try to use the username as fallback, this may or may not be appropriate. Default: blank
- `user`, string. SMTP username. Default: blank
- `password`, string. SMTP password. Leaving both username and password empty the SMTP authentication will be disabled. Default: blank
- `auth_type`, integer. 0 means `Plain`, 1 means `Login`, 2 means `CRAM-MD5`, 3 means `XOAUTH2`. Default: `0`.
- `encryption`, integer. 0 means no encryption, 1 means `TLS`, 2 means `STARTTLS`. Default: `0`.
- `domain`, string. Domain to use for `HELO` command, if empty `localhost` will be used. Default: blank.
- `templates_path`, string. Path to the email templates. This can be an absolute path or a path relative to the config dir. Templates are searched within a subdirectory named "email" in the specified path. You can customize the email templates by simply specifying an alternate path and putting your custom templates there.
- `debug`, integer. Set to `1` to enable SMTP debug. Default: `0`.
- `oauth2`, struct containing OAuth2 related configurations:
  - `provider`, integer, 0 means `Google`, 1 means `Microsoft`. Default: `0`.
  - `tenant`, string. Azure Active Directory tenant for the Microsoft provider. Typical values are `common`, `organizations`, `consumers` or tenant identifier. If empty `common` is used. Default: blank.
  - `client_id`, string. Default: blank.
  - `client_secret`, string. Default: blank.
  - `refresh_token`, string. Default: blank.

## Plugins

The `plugins` section allow to configure a list of external plugins. :warning: Please note that the plugin system is experimental, the configuration parameters and interfaces may change in a backward incompatible way in future. Each plugin is configured using a struct with the following fields:

- `type`, string. Defines the plugin type. Supported types: `notifier`, `kms`, `auth`, `eventsearcher`, `ipfilter`.
- `notifier_options`, struct. Defines the options for notifier plugins.
  - `fs_events`, list of strings. Defines the filesystem events that will be notified to this plugin.
  - `provider_events`, list of strings. Defines the provider events that will be notified to this plugin.
  - `provider_objects`, list if strings. Defines the provider objects that will be notified to this plugin.
  - `log_events`, list of integers. Defines the log events that will be notified to this plugin. `1` means "Login failed", `2` means "Login with non-existent user", `3` means "No login tried", `4` means "Algorithm negotiation failed", `5` means "Login succeeded".
  - `retry_max_time`, integer. Defines the maximum number of seconds an event can be late. SFTPGo adds a timestamp to each event and add to an internal queue any events that a the plugin fails to handle (the plugin returns an error or it is not running). If a plugin fails to handle an event that is too late, based on this configuration, it will be discarded. SFTPGo will try to resend queued events every 30 seconds. 0 means no retry.
  - `retry_queue_max_size`, integer. Defines the maximum number of events that the internal queue can hold. Once the queue is full, the events that cannot be sent to the plugin will be discarded. 0 means no limit.
- `kms_options`, struct. Defines the options for kms plugins.
  - `scheme`, string. KMS scheme. Supported schemes are: `awskms`, `gcpkms`, `hashivault`, `azurekeyvault`.
  - `encrypted_status`, string. Encrypted status for a KMS secret. Supported statuses are: `AWS`, `GCP`, `VaultTransit`, `AzureKeyVault`.
- `auth_options`, struct. Defines the options for auth plugins.
  - `scope`, integer. 1 means passwords only. 2 means public keys only. 4 means key keyboard interactive only. 8 means TLS certificate. The flags can be combined, for example 6 means public keys and keyboard interactive. The scope must be explicit, `0` is not a valid option.
- `cmd`, string. Path to the plugin executable.
- `args`, list of strings. Optional arguments to pass to the plugin executable.
- `sha256sum`, string. SHA256 checksum for the plugin executable. If not empty it will be used to verify the integrity of the executable.
- `auto_mtls`, boolean. If enabled the client and the server automatically negotiate mutual TLS for transport authentication. This ensures that only the original client will be allowed to connect to the server, and all other connections will be rejected. The client will also refuse to connect to any server that isn't the original instance started by the client.
- `env_prefix`, string. Defines the prefix for env vars to pass from the SFTPGo process environment to the plugin. Set to `none` to not pass any environment variable, set to `*` to pass all environment variables. If empty, the prefix is returned as the plugin name in uppercase with `-` replaced with `_` and a trailing `_`. For example if the plugin name is `sftpgo-plugin-eventsearch` the prefix will be `SFTPGO_PLUGIN_EVENTSEARCH_`
- `env_vars`, list of strings. Additional environment variable names to pass from the SFTPGo process environment to the plugin.
