# Environment variables

The configuration file can change between different versions and merging your custom settings with the default configuration file, after updating SFTPGo, may be time-consuming. For this reason we suggest to set your custom settings using environment variables. This eliminates the need to merge your changes with the default configuration file after each update, you have to just check that your custom configuration keys still exists.

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
- To set `command.commands` to `[{"path": "/usr/bin/date"}, {"path: "/usr/bin/ping", "args": ["-c5", "example.com"]}]`, define the env vars:
  - `SFTPGO_COMMAND__COMMANDS__0__PATH` with value `/usr/bin/date`,
  - `SFTPGO_COMMAND__COMMANDS__1__PATH` with value `/usr/bin/ping`, and
  - `SFTPGO_COMMAND__COMMANDS__1__ARGS` with value `-c5,example.com`.

Notice how `command.commands[1].args` is itself a list of strings, so the value of `SFTPGO_COMMAND__COMMANDS__1__ARGS` is a comma-separated list.

## Variable sources

Setting configuration options from environment variables is natural in Docker/Kubernetes.
If you install SFTPGo on Linux using the official deb/rpm packages you can set your custom environment variables in the file `/etc/sftpgo/sftpgo.env` (create this file if it does not exist, it is defined as `EnvironmentFile` in the SFTPGo systemd unit).
SFTPGo also reads files inside the `env.d` directory relative to config dir and then exports the valid variables into environment variables if they are not already set. With this method you can override any configuration options, set environment variables for SFTPGo plugins but you cannot set command flags because these files are read after that SFTPGo starts and the config dir must already be set.
Of course you can also set environment variables with the method provided by the operating system of your choice.

The following escaping rules apply to environment variable files in the `env.d` directory:

- If you use single quotes nothing is escaped.
- If you use double quotes you can escape characters using a backslash (`\`). `$` has special meaning and tries to expand to another environment variable if not escaped.

Suppose you want to set the dataprovider password to `my$secret\pwd`, you can use one of the following formats:

- `SFTPGO_DATA_PROVIDER__PASSWORD='my$secret\pwd'`.
- `SFTPGO_DATA_PROVIDER__PASSWORD="my\$secret\\pwd"`.

## Additional environment variables

Some additional environment variables are available:

- `SFTPGO_HOOK__MEMORY_PIPES__ENABLED` set to `1` to enable memory pipes. This allows fully in-memory transfers to and from cloud storage backends, eliminating the need for temporary disk files.
- `SFTPGO_HOOK__WEBCLIENT_DISABLE_PREVIEW`, set to `1` to disable file preview in the WebClient UI.
- `SFTPGO_HOOK__WEBCLIENT_DISABLE_EDITOR`, set to `1` to disable the built-in text editor in the WebClient UI.
- `SFTPGO_HOOK__S3_CHECK_PARENT_DIR`, set to `1` to prevent uploads to non-existent directories when using S3 backends, emulating the behavior of a local filesystem. By default, uploads to non-existent directories are allowed on cloud storage backends due to their flat structure.
- `SFTPGO_HOOK__GCS_CHECK_PARENT_DIR`, set to `1` to prevent uploads to non-existent directories when using Google Cloud Storage backends.
- `SFTPGO_HOOK__AZBLOB_CHECK_PARENT_DIR`, set to `1` to prevent uploads to non-existent directories when using Azure Blob Storage backends.
- `SFTPGO_HOOK__EXTRACT_MAX_COMPRESSION_RATIO`, sets the maximum allowed compression ratio for extracted ZIP files. Default: `60`.
- `SFTPGO_HOOK__EXTRACT_MAX_FILES`, sets the maximum number of files allowed in a ZIP archive. Default: `1000`.
- `SFTPGO_HOOK__EXTRACT_MAX_SIZE`, sets the maximum total uncompressed size allowed for a ZIP archive in MB. Default: `1024`.
- `SFTPGO_HOOK__PASSWORD_EXPIRATION_EMAIL_SUBJECT`, allows customizing the subject line of the password expiration notification email. Default: `SFTPGo password expiration notification`.
- `SFTPGO_HOOK__PASSWORD_FORGOT_EMAIL_SUBJECT`, allows customizing the subject line of the “forgot password” email sent to users. Default: `Email Verification Code for <username>`.
- `SFTPGO_HOOK__SHARE_CODE_EMAIL_SUBJECT`, allows customizing the subject line of emails containing share codes for shares. Default: `Share access code`.
- `SFTPGO_HOOK__DISABLE_DOT_ENTRIES`, set to `1` to hide `.` and `..` entries from SFTP directory listings.
- `SFTPGO_HOOK__OAUTH2_DISABLE_PKCE`, set to `1` to disable PKCE for OAuth2 authentication flows used by IMAP and SMTP.
- `SFTPGO_HOOK__HAS_SHARE_LEGAL_AGREEMENT`, set to `1` to display a legal agreement can be displayed before granting access to the share by external users.
- `SFTPGO_HOOK_SHARE_LEGAL_TMPL_PATH`, set to the path of a custom HTML template for the share agreement. We recommend using the default template as a starting point, which is typically located at `/usr/share/sftpgo/templates/webclient/sharelegal.html` or an equivalent path depending on the installation method.
- `SFTPGO_HOOK__ENABLE_OIDC_UI`, set to 1 to add the OpenID Connect configuration section for the first binding in the WebAdmin UI. If more than an OpenID Connect configuration is required, use the configuration file or environment variables to override it instead.
- `SFTPGO_HOOK__HTTPD_MAX_REQUEST_SIZE`. Allows configuring the maximum size of HTTP requests in MB. Default: 1 MB. :warning: Increasing this value may expose the server to large payloads, which can impact memory usage or allow denial-of-service attacks.
- `SFTPGO_HOOK__HTTPD_MAX_RESTORE_SIZE`. Allows configuring the maximum size of a backup to restore in MB. Default: 20 MB.
