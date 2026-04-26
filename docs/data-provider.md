---
description: "Initialize and manage the SFTPGo data provider. Supports SQLite, PostgreSQL, MySQL, CockroachDB, and Bolt databases for user and configuration storage."
---

# Data provider initialization and management

**In the default configuration, you do not need to initialize the data provider manually.** SFTPGo creates the schema on first startup and applies any pending migrations on every subsequent startup, provided the configured credentials can issue DDL on the target database. Just point SFTPGo at an empty database (or accept the default SQLite file) and start the service.

What you do need to provide yourself:

- **PostgreSQL, MySQL, CockroachDB**: create the empty database (or schema) and a user with sufficient privileges before starting SFTPGo.
- **SQLite**: the database file is created automatically.
- **Memory and bolt**: no database to create. After an SFTPGo upgrade the existing data may be migrated automatically on first startup.

For step-by-step setup instructions for each database, see [Configuration examples](configuration-examples.md#database-providers).

## Running initprovider manually

`initprovider` is a one-shot command that performs the same schema creation and migration that SFTPGo normally does at startup. You only need it when you have set `update_mode` to `1` — that is, when you explicitly disable automatic checks/updates so that migrations cannot run under the service's own credentials. Common reasons:

- The runtime database account has intentionally limited privileges (no DDL) and a separate ops role owns schema evolution.
- You want to gate upgrades behind a human-approved step rather than letting them run at service startup.

In that mode, run `initprovider` as the privileged role before starting the new version:

```bash
sftpgo initprovider
```

See the available options with:

```bash
sftpgo initprovider --help
```

With `update_mode = 0` (the default) running `initprovider` is harmless — it just performs the same work the next startup would have done — but it is not required.

:warning: Some data providers (e.g. MySQL and CockroachDB) do not support schema changes within a transaction, so a forcibly-aborted migration can leave the schema in an inconsistent state. CockroachDB also does not implement `pg_advisory_lock`, which means migrations cannot be serialized at the database level: make sure only one SFTPGo instance runs migrations against CockroachDB at a time — either by starting instances sequentially, or by setting `update_mode` to `1` on every instance and running `initprovider` from a single operator job.

## Upgrading

SFTPGo supports upgrading from the previous release branch to the current one.
Some examples for supported upgrade paths are:

- from 2.1.x to 2.2.x
- from 2.2.x to 2.3.x and so on.

For supported upgrade paths the data and schema are migrated automatically when SFTPGo starts (or by running `initprovider` manually if you opted into `update_mode = 1`).

So if, for example, you want to upgrade from 2.0.x to 2.2.x, you must first install version 2.1.x, let it update the data provider (automatically at startup, or manually via `initprovider` if you run in manual mode), and finally install version 2.2.x. It is recommended to always install the latest available minor version — do not install 2.1.0 if 2.1.2 is available.

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
