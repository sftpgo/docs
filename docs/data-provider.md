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

If for some reason you want to downgrade SFTPGo, you may need to downgrade your data provider schema and data as well. You can use the `revertprovider` command for this task.

As for upgrading, SFTPGo supports downgrading from the previous release branch to the current one.

So, if you plan to downgrade from 2.3.x to 2.2.x, before uninstalling 2.3.x version, you can prepare your data provider executing the following command from the configuration directory:

```shell
sftpgo revertprovider
```

Take a look at the CLI usage to learn how to specify a configuration file:

```shell
sftpgo revertprovider --help
```

The `revertprovider` command is not supported for the memory provider.

Please note that we only support the current release branch and the current main branch, if you find a bug it is better to report it rather than downgrading to an older unsupported version.
