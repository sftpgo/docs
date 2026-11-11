---
description: "Initialize and manage the open-source SFTPGo data provider. Supports SQLite, PostgreSQL, MySQL, CockroachDB, and Bolt databases for user and configuration storage."
---

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

## Backups and restores

A backup, or dump, is a provider-independent JSON document with the objects of the requested scopes. It is produced by the `dumpdata` REST API, by the "Maintenance" section of the WebAdmin, by the periodic backup and by the backup [event action](eventmanager.md), and restored by the `loaddata` REST API, by the WebAdmin and by the `--loaddata-from` flag of `serve` and `initprovider`.

A backup is built in memory before it is written: plan about ten times the size of the backup it produces, for example around 800 MB for the 70 MB backup of 20,000 users with three virtual folders each and their 60,000 folders. To lower the memory a single backup needs on a large configuration, back up the object types separately with the `scopes` parameter of `dumpdata`, comma separated: `users`, `folders`, `groups`, `admins`, `api_keys`, `shares`, `actions`, `rules`, `roles`, `ip_lists`, `configs`.

A restore accepts a backup of up to 20 MB and decodes it in memory: plan up to about 700 MB for a backup close to that size.

A backup carries up to 60,000 IP list entries, counted across the lists together. Past that the backup fails with an explicit error, so it either carries every entry or does not complete: request the other scopes to back up the rest of the configuration.

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

:warning: The schema downgrade removes the virtual folder [sub-path mount](virtual-folders.md#sub-path-mounts) field (`subpath`): each affected mapping then serves its whole folder, so users gain access to content the sub-path mount kept hidden. Review and remove sub-path mounts before downgrading, or replace them with dedicated folders. A folder mapped at multiple virtual paths by the same user or group makes the downgrade fail: remove the extra mappings first.

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
