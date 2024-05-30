# Environment variables

You can override all the available [configuration options](config-file.md) using environment variables. SFTPGo will check for environment variables with a name matching the key uppercased and prefixed with the `SFTPGO_`. You need to use `__` to traverse a struct.

Let's see some examples:

- To set the `port` for the first sftpd binding, you need to define the env var `SFTPGO_SFTPD__BINDINGS__0__PORT`
- To set the `execute_on` actions, you need to define the env var `SFTPGO_COMMON__ACTIONS__EXECUTE_ON`. For example `SFTPGO_COMMON__ACTIONS__EXECUTE_ON=upload,download`

The configuration file can change between different versions and merging your custom settings with the default configuration file, after updating SFTPGo, may be time-consuming. For this reason we suggest to set your custom settings using environment variables. This eliminates the need to merge your changes with the default configuration file after each update, you have to just check that your custom configuration keys still exists.

Setting configuration options from environment variables is natural in Docker/Kubernetes.
If you install SFTPGo on Linux using the official deb/rpm packages you can set your custom environment variables in the file `/etc/sftpgo/sftpgo.env` (create this file if it does not exist, it is defined as `EnvironmentFile` in the SFTPGo systemd unit).
SFTPGo also reads files inside the `env.d` directory relative to config dir and then exports the valid variables into environment variables if they are not already set. With this method you can override any configuration options, set environment variables for SFTPGo plugins but you cannot set command flags because these files are read after that SFTPGo starts and the config dir must already be set.
Of course you can also set environment variables with the method provided by the operating system of your choice.

:warning: For environment variable files within the `env.d` directory, use single-quoted strings to avoid unexpected substitutions of characters preceded by a dollar sign (`$`).
