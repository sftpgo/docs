# Data provider initialization and management

Before starting the SFTPGo server please make sure that the configured data provider is properly initialized/updated.

For PostgreSQL, MySQL and CockroachDB providers, you need to create the configured database. For SQLite, the configured database will be automatically created at startup. Memory and bolt data providers do not require any initialization but they could require an update to the existing data after upgrading SFTPGo.

SFTPGo will attempt to automatically detect if the data provider is initialized/updated and if not, will attempt to initialize/ update it on startup as needed.

Alternately, you can create/update the required data provider structures yourself using the `initprovider` command.

For example, you can simply execute the following command from the configuration directory:

```bash
sftpgo initprovider
```

Take a look at the CLI usage to learn how to specify a different configuration file:

```bash
sftpgo initprovider --help
```

You can disable automatic data provider checks/updates at startup by setting the `update_mode` configuration key to `1`.

:warning: Please note that some data providers (e.g. MySQL and CockroachDB) do not support schema changes within a transaction, this means that you may end up with an inconsistent schema if migrations are forcibly aborted. CockroachDB doesn't support database-level locks, so make sure you don't execute migrations concurrently.

## Upgrading

SFTPGo supports upgrading from the previous release branch to the current one.
Some examples for supported upgrade paths are:

- from 2.1.x to 2.2.x
- from 2.2.x to 2.3.x and so on.

For supported upgrade paths, the data and schema are migrated automatically when SFTPGo starts, alternatively you can use the `initprovider` command before starting SFTPGo.

So if, for example, you want to upgrade from 2.0.x to 2.2.x, you must first install version 2.1.x, update the data provider (automatically, by starting SFTPGo or manually using the `initprovider` command) and finally install the version 2.2.x. It is recommended to always install the latest available minor version, ie do not install 2.1.0 if 2.1.2 is available.

Loading data from a provider independent JSON dump is supported from the previous release branch to the current one too. After upgrading SFTPGo it is advisable to regenerate the JSON dump from the new version.

## Downgrading

If you need to downgrade SFTPGo to a previous version, simply replacing the binary is not enough. You must also revert the database schema and data to make them compatible with the older version.

Newer versions of SFTPGo often introduce changes to the data provider (database), such as new tables or columns. Older binaries do not know how to handle these changes and will fail to start if the database schema version is higher than what they expect.

To handle this, you can use the `revertprovider` command.

:warning: Before proceeding, strictly backup your database. You can also create a database-independent backup (JSON format) from the "Maintenance" section of the WebAdmin or using the REST API. Downgrading the schema involves dropping tables or columns added in the newer version, resulting in irreversible data loss for those specific fields.

### How to use revertprovider

You must execute this command before uninstalling the current (newer) version of SFTPGo. The current binary contains the logic necessary to revert the changes it applied.

Run the command from your configuration directory:

```shell
sftpgo revertprovider --to-version <VERSION_NUMBER>
```

You can verify the command options and defaults using:

```shell
sftpgo revertprovider --help
```

### Schema Versions

The `--to-version` parameter accepts an integer representing the internal schema version you want to target. The current schema version is 34.

Here are the supported target versions for downgrading:

- **34 (Current)**: The latest schema version.
- **33 (v2.7.x)**: **Default value.** Target this version to downgrade to v2.7.x.
- **30, 31, 32 (Intermediate)**: Intermediate schema versions between v2.6.x and v2.7.x.
- **29 (v2.6.x)**: Target this version to downgrade to v2.6.x.

### Example

If you are currently running the latest version (Schema 34) and want to downgrade to v2.6.x, you must revert the database schema to version 29.

- Stop the SFTPGo service.
- Run the revert command:

```shell
sftpgo revertprovider --to-version 29
```

- Uninstall the current SFTPGo binary.
- Install SFTPGo v2.6.x.
- Start the service.

Note: The `revertprovider` command is not supported (and not needed) for the memory provider, as data is not persisted.

Support Policy: Please note that we officially support only the current release branch and the main branch. If you are downgrading because you found a bug, we strongly encourage you to report the issue so we can fix it, rather than reverting to an older, unsupported version.
