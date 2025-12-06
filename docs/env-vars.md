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
