---
description: "SFTPGo command-line reference: start the server, configure logging, load data, manage the Windows service, and use portable mode."
---

# Command line options

The SFTPGo executable can be used this way:

```console
Usage:
  sftpgo [command]

Available Commands:
  acme           Obtain TLS certificates from ACME-based CAs like Let's Encrypt
  convertnames   Convert existing names to conform to the configured naming rules
  convertsecrets Re-encrypt the stored KMS secrets with a new master key and/or provider
  gen            A collection of useful generators
  help           Help about any command
  initprovider   Initialize and/or updates the configured data provider
  ping           Issues an health check to SFTPGo
  portable       Serve a single directory/account
  resetadmin     Restore access for the specified administrator
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
- `--log-utc-time` boolean. Enable UTC time for logging. Default `false` or the value of `SFTPGO_LOG_UTC_TIME` environment variable (1 or `true`, 0 or `false`).

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

## Restoring access for an administrator

If you lose access to an administrator account you can restore it from the host where SFTPGo is running. The `resetadmin` command connects directly to the data provider, using the configuration file and any `SFTPGO_*` environment variable overrides, and updates the admin record. The flags select what to reset and can be combined:

```shell
sftpgo resetadmin --admin <username> --reset-password
```

prompts interactively for a new password (and a confirmation).

```shell
sftpgo resetadmin --admin <username> --reset-two-factor
```

disables two-factor authentication — useful when the admin knows the password but lost access to the second-factor device. The admin can re-enable 2FA from the WebAdmin after logging in.

```shell
sftpgo resetadmin --admin <username> --reset-restrictions
```

re-enables the administrator, clears its IP allow list and re-enables password authentication — useful when the admin is locked out by a misconfigured allow list, a disabled status, or password authentication turned off (for example an OpenID Connect-only admin whose identity provider is unavailable). Each removed restriction is printed to the console, including the previous allow list value so you can reconfigure it correctly after logging in. The password and the two-factor configuration are preserved.

The legacy `resetpwd` command is kept for backward compatibility and is equivalent to `resetadmin --reset-password --reset-two-factor`.

Notes and limitations:

- **Not supported for the memory data provider** (nothing is persisted).
- **For embedded data providers (bolt, SQLite)** stop the running SFTPGo service first, otherwise the database can become corrupted (a single-writer restriction).
- **For shared data providers (PostgreSQL, MySQL, CockroachDB)** the command can be run while SFTPGo is serving — no downtime required.
- Pass `-c`/`--config-dir` (or `SFTPGO_CONFIG_DIR`) if the config directory is not the current working directory.

On Windows, run the command from an elevated PowerShell (the service account must have access to the data provider):

```powershell
& "C:\Program Files\SFTPGo\sftpgo.exe" resetadmin --admin yourname --reset-password -c "C:\ProgramData\SFTPGo Enterprise"
```

## Converting names to the configured naming rules

The `naming_rules` data provider setting (see [config file](config-file.md)) normalizes usernames, folder, group, role, event action and event rule names before they are saved and looked up. Enabling lowercase or trim on an installation that already holds names which do not conform makes those objects unreachable: the lookup is normalized while the stored row is not. The `convertnames` command reconciles the stored data with the configured rules:

```shell
sftpgo convertnames
```

By default the command only reports the names it would convert and any blocking conflict, and never modifies the data. Review the report, then apply it:

```shell
sftpgo convertnames --execute
```

The `--execute` flag can also be set with the `SFTPGO_CONVERTNAMES_EXECUTE` environment variable.

Each name is converted with a single statement that changes only the name column. The operation is idempotent: if it stops on an error, the names already converted stay committed and you can fix the cause and run it again to convert the rest.

A blocking conflict is reported when two distinct names would collapse to the same value once the rules are applied (for example `Team` and `team` with the lowercase rule), or when the target name is still held by a soft-deleted row pending purge. Resolve these by hand before running `--execute`.

Notes and limitations:

- **Not supported for the memory data provider** (nothing is persisted).
- **Conversion is available for the SQL data providers** (SQLite, PostgreSQL, MySQL/MariaDB, CockroachDB). On bolt the report is produced and the conversion step is skipped.
- :warning: Run the command while the SFTPGo instance is stopped. With SQLite a running server can modify the same database concurrently, and an embedded database must be reachable exclusively by the command.
- With PostgreSQL, MySQL and CockroachDB the default collation is case-insensitive, so mixed-case names keep resolving and converting is portability oriented; it matters when moving to a case-sensitive provider.
- Pass `-c`/`--config-dir` (or `SFTPGO_CONFIG_DIR`) if the config directory is not the current working directory.

## Re-encrypting the stored secrets

The [KMS](kms.md) master key is global and is not stored with the secret: every stored secret is decrypted with the master key currently configured. Changing the master key in the configuration therefore makes the secrets encrypted with the previous key undecryptable. The provider, instead, is recorded per secret, so secrets encrypted by one provider keep being decrypted by it after the configured provider changes. The `convertsecrets` command re-encrypts every stored secret (user, admin, group and folder filesystem credentials, two-factor authentication secrets and recovery codes, event action and configuration secrets) onto a new master key and/or provider in a single pass, so you can rotate the master key or migrate provider without losing access to the data.

Rotate the master key, for example to replace a weak one, keeping the current key in the configuration and passing the new one on the command line:

```shell
sftpgo convertsecrets --new-master-key <new-key>
sftpgo convertsecrets --new-master-key-path /path/to/new-key
```

Migrate the secrets to a different provider, for example between `local` and the default `aes256gcm` in either direction:

```shell
sftpgo convertsecrets --target-provider aes256gcm
```

`--target-provider` also accepts the full provider URL of an external KMS, so you can migrate to or from one as long as the matching KMS plugin is configured (it is loaded before the secrets are touched):

```shell
sftpgo convertsecrets --target-provider "hashivault://my-key"
```

To re-encrypt the secrets without a master key, pass `--no-master-key`. This weakens confidentiality — the secrets become recoverable from a copy of the data store alone — and the command warns accordingly; it is mutually exclusive with `--new-master-key`/`--new-master-key-path`.

The two can be combined. The current key/provider come from the loaded configuration and must be able to decrypt the existing secrets; the new key/provider come from the command line. By default the command only reports what would change; add `--execute` to perform the re-encryption:

```shell
sftpgo convertsecrets --new-master-key <new-key> --execute
```

After the run completes, update the `kms.secrets` configuration on every instance to the new key and/or provider, then restart. Keep the previous key available until you have verified the new configuration works: the backup written before `--execute` contains the secrets encrypted with the **previous** key, so restoring it requires the previous key in the configuration.

:information_source: The KMS master key wraps the secrets stored in the data provider — including the **CryptFs passphrase**, which is itself stored as a KMS secret. `convertsecrets` re-wraps these secrets under the new master key without changing their values, so the CryptFs passphrase keeps deriving the same per-file keys and the encrypted files on disk are not affected. Changing the CryptFs passphrase value itself is a different operation that re-derives the file keys and requires re-uploading the affected files (see [Data At Rest Encryption](dare.md)).

Notes and limitations:

- **Available for the SQL data providers** (SQLite, PostgreSQL, MySQL/MariaDB, CockroachDB). The memory and bolt providers are outside the scope of the command.
- :warning: With the embedded SQLite database, run the command while the SFTPGo instance is stopped: a running server can modify the same database file concurrently, and the file must be reachable exclusively by the command. With a networked SQL database (PostgreSQL, MySQL/MariaDB, CockroachDB) the command can run while the server, or the whole cluster, is live.
- The operation is resumable and idempotent: an interrupted run can simply be re-executed, and secrets already on the target key and provider are detected and left unchanged. This holds for both a master-key rotation and a provider migration.
- On a networked SQL database (PostgreSQL/MySQL/CockroachDB) the command can run while the server is live, whether it is a single instance or a cluster, but without a [master key ring](#rotating-the-master-key-without-downtime) a running instance still on the previous key can no longer decrypt the secrets already moved to the new key, so the credentials that need them in plain text fail until it is reconfigured and restarted. Configure the ring first, so the running instances can decrypt both keys while the secrets are migrated.
- Unless `--skip-backup` is given, `--execute` first writes a full data provider backup (the same content as `dumpdata`) to a new file and never overwrites an existing one. Use `--backup-file` to choose the path.
- A backup is strongly recommended before re-encrypting. Pass `--skip-backup` only if you maintain backups by other means. Restoring the backup with `loaddata` re-validates every entity, so entities that were valid when stored but no longer satisfy the current rules (for example a username that predates a stricter naming rule) may be rejected on restore even though `convertsecrets` itself preserves them.
- Pass `-c`/`--config-dir` (or `SFTPGO_CONFIG_DIR`) if the config directory is not the current working directory.

### Rotating the master key without downtime

With the embedded SQLite database the simplest path is to stop the server, run `convertsecrets`, then start with the new key. On a networked SQL database (PostgreSQL, MySQL/MariaDB, CockroachDB) you can rotate the master key with no downtime using the [master key ring](config-file.md#master-key-ring), whether the deployment is a single instance or a cluster: a ring with the old key and the new key lets the running instances decrypt the secrets the rotation is still moving from one key to the other.

The ring is the active key (`master_key`/`master_key_path`) plus the decrypt-only keys (`additional_master_keys`/`additional_master_keys_path`). Keeping `OLD` in the ring as decrypt-only while `NEW` encrypts lets the running instances decrypt the secrets the rotation is still moving:

1. **Configure the ring.** Set `NEW` active and keep `OLD` as a decrypt-only key (`master_key: NEW`, `additional_master_keys: [OLD]`) and apply it with a restart. New writes are now encrypted with `NEW`, while existing `OLD` secrets stay readable through the decrypt-only key.
2. **Re-encrypt.** Run `sftpgo convertsecrets --new-master-key <NEW> --execute`. It moves the remaining `OLD` secrets to `NEW` and can run while the deployment is live; no new `OLD` secrets are produced because the instances already encrypt with `NEW`.
3. **Retire the old key.** Run `sftpgo convertsecrets --new-master-key <NEW>` (without `--execute`) and confirm it reports nothing left to convert, then remove `OLD` from `additional_master_keys` so only `NEW` is configured and apply it. The old key is gone.

:information_source: In a cluster rolled one instance at a time, stage `NEW` as decrypt-only *before* step 1: roll out `master_key: OLD`, `additional_master_keys: [NEW]` to every instance first, so an instance still on the previous configuration can decrypt a secret another instance has already written with `NEW`. Promote `NEW` to active (step 1) only once every instance holds it.

The two-key case above is the common one. The ring can hold more than two keys, so a second rotation can start before the previous old key is retired (`master_key: NEW2`, `additional_master_keys: [NEW1, OLD]`), and a store that accumulated secrets under several historical keys can be consolidated by keeping every key still needed in `additional_master_keys` until a single `convertsecrets` run moves them all onto the active key. A dry run reporting nothing left to convert is the signal that a decrypt-only key can be dropped.

:warning: While the ring holds more than one key, a compromise of **any** key in the ring exposes the secrets. Keep the window short and remove the decrypt-only key as soon as step 3 confirms the migration is complete.

## Other commands

For other commands run `sftpgo <command> --help` to understand the usage.
