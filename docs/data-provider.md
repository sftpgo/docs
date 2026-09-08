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

## Loading a dump from the command line

`initprovider` can restore a provider dump — the JSON produced by the `dumpdata` REST API or from the "Maintenance" section of the WebAdmin — and then exit, so an instance can be seeded or refreshed without starting the service. The schema work runs first, so a single command initializes an empty database and populates it:

```bash
sftpgo initprovider --loaddata-from /path/to/backup.json
```

The path must be absolute. Each flag can also be set through the matching environment variable:

| Flag | Environment variable | Description |
| ---- | -------------------- | ----------- |
| `--loaddata-from` | `SFTPGO_LOADDATA_FROM` | Absolute path of the dump to restore. |
| `--loaddata-mode` | `SFTPGO_LOADDATA_MODE` | `0` adds new objects and updates existing ones, `1` adds new objects and keeps existing ones as they are. Default `1`. |
| `--loaddata-clean` | `SFTPGO_LOADDATA_CLEAN` | Delete the file after a successful load. Default `false`. |
| `--loaddata-prune` | `SFTPGO_LOADDATA_PRUNE` | Delete the objects the dump does not contain. Default `false`. |

A restore accepts a dump of up to 20 MB. For a larger file raise the limit with the `SFTPGO_HOOK__HTTPD_MAX_RESTORE_SIZE` environment variable, described under [HTTP server limits](env-vars.md#http-server-limits), on the instance that performs the restore: the same limit applies to the REST API and the WebAdmin.

:warning: A dump holds the whole configuration, including password hashes and the KMS-encrypted secrets, so protect the file and the channel that carries it as you would the database itself.

The command loads the data and exits, so it starts no scheduler and runs no quota scan: use `serve --loaddata-scan` when the restored users need their quota recalculated. It leaves the data provider [actions hook](custom-actions.md#provider-events) alone for the same reason.

### Restoring a dump as a mirror

`--loaddata-prune` deletes the objects that are missing from the dump, so the types it carries end up matching it. It is meant for a standby instance that follows a primary: every cycle downloads a fresh dump and applies it, and the objects deleted on the primary are deleted locally too.

```bash
sftpgo initprovider --loaddata-from /path/to/backup.json --loaddata-mode 0 --loaddata-prune
```

Users, admins, groups, folders, roles, event actions, event rules, shares, API keys and IP list entries are pruned. Configurations and the license are single objects and are kept.

Two conditions apply:

- **`--loaddata-mode 0` is required.** Mode 1 keeps existing objects as they are, so pruning would delete what the dump does not carry while leaving a stale copy of everything else. The combination is rejected.
- **The dump records the scopes it carries**, and only those types are pruned. A partial dump obtained with `GET /dumpdata?scopes=users,folders` prunes users and folders and leaves everything else in place. A dump produced by a version that predates this field is refused, because a partial dump cannot be told apart from a complete one, and so is a dump that declares the admins scope while carrying no admin.

:warning: The instance is a copy of the dump, so treat its data provider as read-only. An object created or edited locally is deleted or overwritten by the next run.

:warning: The dump carries the KMS-encrypted secrets as they are stored and the restore writes them unchanged, so the instance must use the same [KMS](kms.md) configuration as the one that produced the dump. With a different key the configuration is replicated faithfully and the secrets it depends on cannot be read. The license is decrypted and re-encrypted during the restore, so when the key is missing the load stops there, with an error that names the license rather than the key.

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

:warning: Reverting to schema version 34 or lower removes the virtual folder [sub-path mount](virtual-folders.md#sub-path-mounts) fields (`subpath` and `exposed_subpaths`): each affected mapping then serves its whole folder, so users gain access to content the sub-path mounts kept hidden. Review and remove sub-path mounts before downgrading, or replace them with dedicated folders. A folder mapped at multiple virtual paths by the same user or group makes the downgrade fail: remove the extra mappings first.

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

The `--to-version` parameter accepts an integer representing the internal schema version you want to target. The current schema version is 36.

Here are the supported target versions for downgrading:

- **36 (Current)**: The latest schema version.
- **34, 35 (Intermediate)**: Intermediate schema versions after v2.7.x.
- **33 (v2.7.x)**: **Default value.** Target this version to downgrade to v2.7.x.
- **30, 31, 32 (Intermediate)**: Intermediate schema versions between v2.6.x and v2.7.x.
- **29 (v2.6.x)**: Target this version to downgrade to v2.6.x.

### Example

If you are currently running the latest version (Schema 36) and want to downgrade to v2.6.x, you must revert the database schema to version 29.

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
