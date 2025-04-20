# Command line options

The SFTPGo executable can be used this way:

```console
Usage:
  sftpgo [command]

Available Commands:
  acme           Obtain TLS certificates from ACME-based CAs like Let's Encrypt
  gen            A collection of useful generators
  help           Help about any command
  initprovider   Initialize and/or updates the configured data provider
  ping           Issues an health check to SFTPGo
  portable       Serve a single directory/account
  resetprovider  Reset the configured provider, any data will be lost
  resetpwd       Reset the password for the specified administrator
  revertprovider Revert the configured data provider to a previous version
  serve          Start the SFTPGo service
  smtptest       Test the SMTP configuration

Flags:
  -h, --help      help for sftpgo
  -v, --version

Use "sftpgo [command] --help" for more information about a command.
```

## Starting the server

To start the SFTPGo server you can use the `serve` command. It supports the following flags:

- `--config-dir` string. Location of the config dir. This directory is used as the base for files with a relative path, e.g. the private keys for the SFTP server or the database file if you use a file-based data provider.. The configuration file, if not explicitly set, is looked for in this dir. We support reading from JSON, TOML, YAML, HCL, envfile and Java properties config files. The default config file name is `sftpgo` and therefore `sftpgo.json`, `sftpgo.yaml` and so on are searched. The default value is the working directory (".") or the value of `SFTPGO_CONFIG_DIR` environment variable.
- `--config-file` string. This flag explicitly defines the path, name and extension of the config file. If must be an absolute path or a path relative to the configuration directory. The specified file name must have a supported extension (JSON, YAML, TOML, HCL or Java properties). The default value is empty or the value of `SFTPGO_CONFIG_FILE` environment variable.
- `--grace-time`, integer. Graceful shutdown is an option to initiate a shutdown without abrupt cancellation of the currently ongoing client-initiated transfer sessions. This grace time defines the number of seconds allowed for existing transfers to get completed before shutting down. 0 means disabled. The default value is `0` or the value of `SFTPGO_GRACE_TIME` environment variable. A graceful shutdown is triggered by an interrupt signal or by a service `stop` request on Windows, if a grace time is configured.
- `--loaddata-from` string. Load users and folders from this file. The file must be specified as absolute path and it must contain a backup obtained using the `dumpdata` REST API or compatible content. The default value is empty or the value of `SFTPGO_LOADDATA_FROM` environment variable.
- `--loaddata-clean` boolean. Determine if the loaddata-from file should be removed after a successful load. Default `false` or the value of `SFTPGO_LOADDATA_CLEAN` environment variable (1 or `true`, 0 or `false`).
- `--loaddata-mode`, integer. Restore mode for data to load. 0 means new users are added, existing users are updated. 1 means new users are added, existing users are not modified. Default 1 or the value of `SFTPGO_LOADDATA_MODE` environment variable.
- `--loaddata-scan`, integer. Quota scan mode after data load. 0 means no quota scan. 1 means quota scan. 2 means scan quota if the user has quota restrictions. Default 0 or the value of `SFTPGO_LOADDATA_QUOTA_SCAN` environment variable.
- `--log-compress` boolean. Determine if the rotated log files should be compressed using gzip. Default `false` or the value of `SFTPGO_LOG_COMPRESS` environment variable (1 or `true`, 0 or `false`). It is unused if `log-file-path` is empty.
- `--log-file-path` string. Location for the log file, default "sftpgo.log" or the value of `SFTPGO_LOG_FILE_PATH` environment variable. Leave empty to write logs to the standard error.
- `--log-max-age` int. Maximum number of days to retain old log files. Default 28 or the value of `SFTPGO_LOG_MAX_AGE` environment variable. It is unused if `log-file-path` is empty.
- `--log-max-backups` int. Maximum number of old log files to retain. Default 5 or the value of `SFTPGO_LOG_MAX_BACKUPS` environment variable. It is unused if `log-file-path` is empty.
- `--log-max-size` int. Maximum size in megabytes of the log file before it gets rotated. Default 10 or the value of `SFTPGO_LOG_MAX_SIZE` environment variable. It is unused if `log-file-path` is empty.
- `--log-level` string. Set the log level. Supported values: `debug`, `info`, `warn`, `error`. Default `debug` or the value of `SFTPGO_LOG_LEVEL` environment variable.
- `--log-utc-time` boolean. Enable UTC time for logging. Default `false` or the value of `SFTPGO_LOG_UTC_TIME` environment variable (1 or `true`, 0 or `false`)

Log file can be rotated on demand sending a `SIGUSR1` signal on Unix based systems and using the command `sftpgo service rotatelogs` on Windows.

## Portable mode

SFTPGo allows to share a single directory on demand using the `portable` command:

```console
sftpgo portable --help
To serve the current working directory with auto generated credentials simply
use:

$ sftpgo portable

Please take a look at the usage below to customize the serving parameters

Usage:
  sftpgo portable [flags]

Flags:
      --allowed-patterns stringArray    Allowed file patterns case insensitive.
                                        The format is:
                                        /dir::pattern1,pattern2.
                                        For example: "/somedir::*.jpg,a*b?.png"
      --az-access-tier string           Leave empty to use the default
                                        container setting
      --az-account-key string
      --az-account-name string
      --az-container string
      --az-download-concurrency int     How many parts are downloaded in
                                        parallel (default 5)
      --az-download-part-size int       The buffer size for multipart downloads
                                        (MB) (default 5)
      --az-endpoint string              Leave empty to use the default:
                                        "blob.core.windows.net"
      --az-key-prefix string            Allows to restrict access to the
                                        virtual folder identified by this
                                        prefix and its contents
      --az-sas-url string               Shared access signature URL
      --az-upload-concurrency int       How many parts are uploaded in
                                        parallel (default 5)
      --az-upload-part-size int         The buffer size for multipart uploads
                                        (MB) (default 5)
      --az-use-emulator
  -c, --config-dir string               Location of the config dir. This directory
                                        is used as the base for files with a relative
                                        path, e.g. the private keys for the SFTP
                                        server or the database file if you use a
                                        file-based data provider.
                                        The configuration file, if not explicitly set,
                                        is looked for in this dir. We support reading
                                        from JSON, TOML, YAML, HCL, envfile and Java
                                        properties config files. The default config
                                        file name is "sftpgo" and therefore
                                        "sftpgo.json", "sftpgo.yaml" and so on are
                                        searched.
                                        This flag can be set using SFTPGO_CONFIG_DIR
                                        env var too. (default ".")
      --config-file string              Path to SFTPGo configuration file.
                                        This flag explicitly defines the path, name
                                        and extension of the config file. If must be
                                        an absolute path or a path relative to the
                                        configuration directory. The specified file
                                        name must have a supported extension (JSON,
                                        YAML, TOML, HCL or Java properties).
                                        This flag can be set using SFTPGO_CONFIG_FILE
                                        env var too.
      --crypto-passphrase string        Passphrase for encryption/decryption
      --denied-patterns stringArray     Denied file patterns case insensitive.
                                        The format is:
                                        /dir::pattern1,pattern2.
                                        For example: "/somedir::*.jpg,a*b?.png"
  -d, --directory string                Path to the directory to serve.
                                        This can be an absolute path or a path
                                        relative to the current directory
                                         (default ".")
  -f, --fs-provider string              osfs => local filesystem (legacy value: 0)
                                        s3fs => AWS S3 compatible (legacy: 1)
                                        gcsfs => Google Cloud Storage (legacy: 2)
                                        azblobfs => Azure Blob Storage (legacy: 3)
                                        cryptfs => Encrypted local filesystem (legacy: 4)
                                        sftpfs => SFTP (legacy: 5) (default "osfs")
      --ftpd-cert string                Path to the certificate file for FTPS
      --ftpd-key string                 Path to the key file for FTPS
      --ftpd-port int                   0 means a random unprivileged port,
                                        < 0 disabled (default -1)
      --gcs-automatic-credentials int   0 means explicit credentials using
                                        a JSON credentials file, 1 automatic
                                         (default 1)
      --gcs-bucket string
      --gcs-credentials-file string     Google Cloud Storage JSON credentials
                                        file
      --gcs-key-prefix string           Allows to restrict access to the
                                        virtual folder identified by this
                                        prefix and its contents
      --gcs-storage-class string
      --grace-time int                  This grace time defines the number of
                                        seconds allowed for existing transfers
                                        to get completed before shutting down.
                                        A graceful shutdown is triggered by an
                                        interrupt signal.

  -h, --help                            help for portable
      --httpd-cert string               Path to the certificate file for WebClient
                                        over HTTPS
      --httpd-key string                Path to the key file for WebClient over
                                        HTTPS
      --httpd-port int                  0 means a random unprivileged port,
                                        < 0 disabled (default -1)
  -l, --log-file-path string            Leave empty to disable logging
      --log-level string                Set the log level.
                                        Supported values:

                                        debug, info, warn, error.
                                         (default "debug")
      --log-utc-time                    Use UTC time for logging
  -p, --password string                 Leave empty to use an auto generated
                                        value
      --password-file string            Read the password from the specified
                                        file path. Leave empty to use an auto
                                        generated value
  -g, --permissions strings             User's permissions. "*" means any
                                        permission (default [list,download])
  -k, --public-key strings
      --s3-access-key string
      --s3-access-secret string
      --s3-acl string
      --s3-bucket string
      --s3-endpoint string
      --s3-force-path-style             Force path style bucket URL
      --s3-key-prefix string            Allows to restrict access to the
                                        virtual folder identified by this
                                        prefix and its contents
      --s3-region string
      --s3-role-arn string
      --s3-skip-tls-verify              If enabled the S3 client accepts any TLS
                                        certificate presented by the server and
                                        any host name in that certificate.
                                        In this mode, TLS is susceptible to
                                        man-in-the-middle attacks.
                                        This should be used only for testing.

      --s3-storage-class string
      --s3-upload-concurrency int       How many parts are uploaded in
                                        parallel (default 2)
      --s3-upload-part-size int         The buffer size for multipart uploads
                                        (MB) (default 5)
      --sftp-buffer-size int            The size of the buffer (in MB) to use
                                        for transfers. By enabling buffering,
                                        the reads and writes, from/to the
                                        remote SFTP server, are split in
                                        multiple concurrent requests and this
                                        allows data to be transferred at a
                                        faster rate, over high latency networks,
                                        by overlapping round-trip times
      --sftp-disable-concurrent-reads   Concurrent reads are safe to use and
                                        disabling them will degrade performance.
                                        Disable for read once servers
      --sftp-endpoint string            SFTP endpoint as host:port for SFTP
                                        provider
      --sftp-fingerprints strings       SFTP fingerprints to verify remote host
                                        key for SFTP provider
      --sftp-key-path string            SFTP private key path for SFTP provider
      --sftp-password string            SFTP password for SFTP provider
      --sftp-prefix string              SFTP prefix allows restrict all
                                        operations to a given path within the
                                        remote SFTP server
      --sftp-username string            SFTP user for SFTP provider
  -s, --sftpd-port int                  0 means a random unprivileged port,
                                        < 0 disabled
      --ssh-commands strings            SSH commands to enable.
                                        "*" means any supported SSH command
                                        including scp
                                         (default [md5sum,sha1sum,sha256sum,cd,pwd,scp])
      --start-directory string          Alternate start directory.
                                        This is a virtual path not a filesystem
                                        path (default "/")
  -u, --username string                 Leave empty to use an auto generated
                                        value
      --webdav-cert string              Path to the certificate file for WebDAV
                                        over HTTPS
      --webdav-key string               Path to the key file for WebDAV over
                                        HTTPS
      --webdav-port int                 0 means a random unprivileged port,
                                        < 0 disabled (default -1)
```

In portable mode you can apply further customizations using a configuration file/environment variables as for the service mode.
SFTP, FTP, HTTP and WebDAV settings configured using the CLI flags are applied to the first binding, any additional bindings will not be affected.

## Manage Windows Service

On Windows, you can register SFTPGo as Windows Service. Take a look at the CLI usage to learn how to do this:

```powershell
PS> sftpgo.exe service --help
Manage SFTPGo Windows Service

Usage:
  sftpgo service [command]

Available Commands:
  install     Install SFTPGo as Windows Service
  reload      Reload the SFTPGo Windows Service sending a "paramchange" request
  rotatelogs  Signal to the running service to rotate the logs
  start       Start SFTPGo Windows Service
  status      Retrieve the status for the SFTPGo Windows Service
  stop        Stop SFTPGo Windows Service
  uninstall   Uninstall SFTPGo Windows Service

Flags:
  -h, --help   help for service

Use "sftpgo service [command] --help" for more information about a command.
```

The `install` subcommand accepts the same flags that are valid for `serve`.

After installing as a Windows Service, please remember to allow network access to the SFTPGo executable using something like this:

```powershell
PS> netsh advfirewall firewall add rule name="SFTPGo Service" dir=in action=allow program="C:\Program Files\SFTPGo\sftpgo.exe"
```

Or through the Windows Firewall GUI.

The Windows installer will register the service and allow network access for it automatically.

## Other commands

For other commands run `sftpgo <command> --help` to understand the usage.
