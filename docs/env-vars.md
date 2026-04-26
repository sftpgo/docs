---
description: "Configure SFTPGo using environment variables. Override any configuration option without editing the config file — ideal for Docker and Kubernetes."
---

# Environment variables

The configuration file can change between different versions and merging your custom settings with the default configuration file, after updating SFTPGo, may be time-consuming. For this reason we suggest setting your custom options using environment variables. This eliminates the need to merge your changes with the default configuration file after each update; you only need to check that your custom configuration keys still exist.

You can override all the available [configuration options](config-file.md) using environment variables.

## Syntax

**SFTPGo will check for environment variables with a name matching the key uppercased and prefixed with `SFTPGO_`. Use `__` to traverse into a struct.** For example:

- To set `common.proxy_protocol` to `1`, define the env var `SFTPGO_COMMON__PROXY_PROTOCOL` with value `1`.
- To set `webdavd.cors.enabled` to `true`, define the env var `SFTPGO_WEBDAVD__CORS__ENABLED` with value `true`.

For options that are list-like, use the following syntax depending on the item type:

**If the option is a list of simple values (booleans, numbers, strings), the list items should be comma-separated.** For example:

- To set `common.actions.execute_on` to `["upload", "download"]`, define the env var `SFTPGO_COMMON__ACTIONS__EXECUTE_ON` with value `upload,download`.
- To set `common.event_manager.enabled_commands` to `["/usr/bin/touch", "/usr/bin/mkdir", "/usr/bin/rm"]`, define the env var `SFTPGO_COMMON__EVENT_MANAGER__ENABLED_COMMANDS` with value `/usr/bin/touch,/usr/bin/mkdir,/usr/bin/rm`.

**If the option is a list of structs, set each struct field with separate env vars, using the list index as the key to traverse into each item struct.** For example:

- To set `sftpd.bindings[0].port` to `22`, define the env var `SFTPGO_SFTPD__BINDINGS__0__PORT` with value `22`.
- To set `command.commands` to `[{"path": "/usr/bin/date"}, {"path": "/usr/bin/ping", "args": ["-c5", "example.com"]}]`, define the env vars:
  - `SFTPGO_COMMAND__COMMANDS__0__PATH` with value `/usr/bin/date`,
  - `SFTPGO_COMMAND__COMMANDS__1__PATH` with value `/usr/bin/ping`, and
  - `SFTPGO_COMMAND__COMMANDS__1__ARGS` with value `-c5,example.com`.

Notice how `command.commands[1].args` is itself a list of strings, so the value of `SFTPGO_COMMAND__COMMANDS__1__ARGS` is a comma-separated list.

## Variable sources

Setting configuration options from environment variables is natural in Docker/Kubernetes.
If you install SFTPGo on Linux using the official deb/rpm packages you can set your custom environment variables in the file `/etc/sftpgo/sftpgo.env` (create this file if it does not exist, it is defined as `EnvironmentFile` in the SFTPGo systemd unit).
SFTPGo also reads files inside the `env.d` directory relative to config dir and then exports the valid variables into environment variables if they are not already set. With this method you can:

- Override any configuration option.
- Set environment variables for SFTPGo plugins.

However, you cannot set command flags this way because these files are read after SFTPGo starts and the config dir must already be set.
Of course you can also set environment variables with the method provided by the operating system of your choice.

**Example:** to enable the SFTP service on port 2222, set the proxy protocol, and configure an EventStore plugin, create a file such as `/etc/sftpgo/env.d/custom.env` with the following content:

```shell
SFTPGO_SFTPD__BINDINGS__0__PORT=2222
SFTPGO_COMMON__PROXY_PROTOCOL=1
SFTPGO_PLUGIN_EVENTSTORE_DSN='host=127.0.0.1 port=5432 dbname=sftpgo_events user=sftpgo password=secret'
SFTPGO_PLUGINS__0__TYPE=notifier
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__FS_EVENTS=upload,download,delete,rename,mkdir,rmdir
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__PROVIDER_EVENTS=add,update,delete
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__PROVIDER_OBJECTS=user,admin,share,event_rule
SFTPGO_PLUGINS__0__CMD=/usr/bin/sftpgo-plugin-eventstore
SFTPGO_PLUGINS__0__ARGS=serve,--driver,postgres
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

The following escaping rules apply to environment variable files in the `env.d` directory:

- If you use single quotes nothing is escaped.
- If you use double quotes you can escape characters using a backslash (`\`). `$` has special meaning and tries to expand to another environment variable if not escaped.

Suppose you want to set the dataprovider password to `my$secret\pwd`, you can use one of the following formats:

- `SFTPGO_DATA_PROVIDER__PASSWORD='my$secret\pwd'`.
- `SFTPGO_DATA_PROVIDER__PASSWORD="my\$secret\\pwd"`.

## Additional environment variables

Some additional environment variables are available, grouped by area.

### Transfers and storage

- `SFTPGO_HOOK__MEMORY_PIPES__ENABLED`, set to `1` to enable memory pipes. This allows fully in-memory transfers to and from cloud storage backends, eliminating the need for temporary disk files.
- `SFTPGO_HOOK__DISABLE_DOT_ENTRIES`, set to `1` to hide `.` and `..` entries from SFTP directory listings.
- `SFTPGO_HOOK__AUTO_FOLDERS`, set to `1` to automatically create virtual folders based on the reply from pre-login and pre-auth hooks. This will cause a database upsert for each returned folder.

### Cloud storage backends

- `SFTPGO_HOOK__S3_CHECK_PARENT_DIR`, set to `1` to prevent uploads to non-existent directories when using S3 backends, emulating the behavior of a local filesystem. By default, uploads to non-existent directories are allowed on cloud storage backends due to their flat structure.
- `SFTPGO_HOOK__GCS_CHECK_PARENT_DIR`, set to `1` to prevent uploads to non-existent directories when using Google Cloud Storage backends.
- `SFTPGO_HOOK__AZBLOB_CHECK_PARENT_DIR`, set to `1` to prevent uploads to non-existent directories when using Azure Blob Storage backends.
- `SFTPGO_HOOK__AZBLOB__DISABLE_CHECKSUM`, set to `1` to disable the CRC64 transactional checksum sent on every Azure Blob upload. Useful for emulators or gateways that do not accept the `x-ms-content-crc64` header. By default the checksum is enabled so the Azure service can verify each uploaded block.
- `SFTPGO_HOOK__GCS_TRUSTED_JSON_CREDENTIALS`, set to `1` to allow the use of any JSON-based credential file for Google Cloud Storage (GCS) backends. When enabled, SFTPGo will accept JSON credentials beyond the standard service account formats, useful in trusted environments. Use with caution, as this bypasses stricter validation.

### WebClient UI

- `SFTPGO_HOOK__WEBCLIENT_DISABLE_PREVIEW`, set to `1` to disable file preview in the WebClient UI.
- `SFTPGO_HOOK__WEBCLIENT_DISABLE_EDITOR`, set to `1` to disable the built-in text editor in the WebClient UI.
- `SFTPGO_HOOK__HAS_SHARE_LEGAL_AGREEMENT`, set to `1` to display a legal agreement before granting access to the share to external users.
- `SFTPGO_HOOK__SHARE_LEGAL_TMPL_PATH`, set to the path of a custom HTML template for the share agreement. We recommend using the default template as a starting point, which is typically located at `/usr/share/sftpgo/templates/webclient/sharelegal.html` or an equivalent path depending on the installation method.

### HTTP server limits

- `SFTPGO_HOOK__HTTPD_MAX_REQUEST_SIZE`, allows configuring the maximum size of HTTP requests in MB. Default: 1 MB. :warning: Increasing this value may expose the server to large payloads, which can impact memory usage or allow denial-of-service attacks.
- `SFTPGO_HOOK__HTTPD_MAX_RESTORE_SIZE`, allows configuring the maximum size of a backup to restore in MB. Default: 20 MB.
- `SFTPGO_HOOK__HTTPD_MAX_EDIT_SIZE`, override the maximum file size viewable in the built-in web editor. Range: 1–10 MB. Default: 2 MB.

### ZIP extraction limits

- `SFTPGO_HOOK__EXTRACT_MAX_COMPRESSION_RATIO`, sets the maximum allowed compression ratio for extracted ZIP files. Default: `60`.
- `SFTPGO_HOOK__EXTRACT_MAX_FILES`, sets the maximum number of files allowed in a ZIP archive. Default: `1000`.
- `SFTPGO_HOOK__EXTRACT_MAX_SIZE`, sets the maximum total uncompressed size allowed for a ZIP archive in MB. Default: `1024`.

### Email subjects

- `SFTPGO_HOOK__PASSWORD_EXPIRATION_EMAIL_SUBJECT`, allows customizing the subject line of the password expiration notification email. Default: `SFTPGo password expiration notification`.
- `SFTPGO_HOOK__PASSWORD_FORGOT_EMAIL_SUBJECT`, allows customizing the subject line of the “forgot password” email sent to users. Default: `Email Verification Code for <username>`.
- `SFTPGO_HOOK__SHARE_CODE_EMAIL_SUBJECT`, allows customizing the subject line of emails containing share codes for shares. Default: `Share access code`.

### Authentication

- `SFTPGO_HOOK__OAUTH2_DISABLE_PKCE`, set to `1` to disable PKCE for OAuth2 authentication flows used by IMAP and SMTP.
- `SFTPGO_HOOK__ENABLE_OIDC_UI`, set to `1` to add the OpenID Connect configuration section for the first binding in the WebAdmin UI. If more than one OpenID Connect configuration is required, use the configuration file or environment variables to override it instead.
- `SFTPGO_HOOK__ENABLE_TLS_UI`, set to `1` to add the TLS certificate configuration section in the WebAdmin UI. Allows uploading a certificate and private key that will be used as the default TLS certificate for the selected protocols (HTTPS, FTPS, WebDAV). Mutually exclusive with automatic certificates (Let's Encrypt/ACME).

### Event manager

- `SFTPGO_HOOK__EVENT_REPORT_MAX_RESULTS`, sets the maximum number of events loaded into memory when generating an event report. This is a server-side safety limit to prevent excessive memory usage regardless of the configured time window. Default: `10000`.
