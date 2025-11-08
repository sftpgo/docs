# Environment variables

You can override all the available [configuration options](config-file.md) and [command line options](cli.md) using environment variables. SFTPGo will check for environment variables with a name matching the key uppercased and prefixed with the `SFTPGO_`. You need to use `__` to traverse a struct.

Let's see some examples:

- To set the `port` for the first sftpd binding, you need to define the env var `SFTPGO_SFTPD__BINDINGS__0__PORT`
- To set the `execute_on` actions, you need to define the env var `SFTPGO_COMMON__ACTIONS__EXECUTE_ON`. For example `SFTPGO_COMMON__ACTIONS__EXECUTE_ON=upload,download`

The configuration file can change between different versions and merging your custom settings with the default configuration file, after updating SFTPGo, may be time-consuming. For this reason we suggest to set your custom settings using environment variables. This eliminates the need to merge your changes with the default configuration file after each update, you have to just check that your custom configuration keys still exists.

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
