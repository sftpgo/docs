---
description: "SFTPGo plugin system: extend functionality with audit logs, LDAP authentication, GeoIP filtering, cloud KMS, and pub/sub event forwarding."
---

# Plugin system

SFTPGo's plugins are completely separate, standalone applications that SFTPGo executes and communicates with over RPC. This means the plugin process does not share the same memory space as SFTPGo and therefore can only access the interfaces and arguments given to it. This also means a crash in a plugin can not crash the entirety of SFTPGo.

:information_source: plugins are trusted components and their responses are applied as given, see [Trust model](external-auth.md#trust-model).

## Configuration

The plugins are configured via the `plugins` section in the main SFTPGo configuration file. You basically have to set the path to your plugin, the plugin type and any plugin specific configuration parameters. If you set the SHA256 checksum for the plugin executable, SFTPGo will verify the plugin integrity before starting it.

For added security you can enable the automatic TLS. In this way, the client and the server automatically negotiate mutual TLS for transport authentication. This ensures that only the original client will be allowed to connect to the server, and all other connections will be rejected. The client will also refuse to connect to any server that isn't the original instance started by the client.

### Plugin indexing

Plugins are configured as a numbered list using environment variables with sequential indices (`SFTPGO_PLUGINS__0__*`, `SFTPGO_PLUGINS__1__*`, `SFTPGO_PLUGINS__2__*`, etc.) or as an array in the configuration file. Each plugin must have a unique index, and indices must be sequential starting from `0`.

For example, if you configure EventStore at index `0` and EventSearch at index `1`, any additional plugin (e.g., KMS, GeoIP) must use index `2` or higher. Using an already occupied index will overwrite the previous plugin configuration.

The following plugin types are supported:

- `auth`, authenticate users via an external service.
- `notifier`, receive notifications for filesystem events (uploads, downloads, deletes, etc.) and provider events (object add, update, delete).
- `kms`, add support for additional Key Management Service providers.
- `eventsearcher`, store and query filesystem and provider events for search and reporting.
- `ipfilter`, allow or deny access based on client IP address.
- `syncer`, synchronize data with external systems.

Full configuration details can be found [here](config-file.md).

## Available plugins

SFTPGo Enterprise includes the following plugins:

- [Audit Logs (EventStore + EventSearch)](plugins/audit-logs.md) — store and search filesystem, provider, and log events in a database.
- [LDAP / Active Directory Authentication](plugins/ldap-auth.md) — authenticate users against LDAP directories, with group mapping support.
- [GeoIP Filtering](plugins/geoip-filter.md) — accept or deny connections based on geographic location.
- [Cloud KMS Providers](plugins/kms-providers.md) — encrypt secrets using Google Cloud KMS, AWS KMS, Azure Key Vault, HashiCorp Vault, or Oracle Key Vault.
- [Pub/Sub Event Forwarding](plugins/pubsub.md) — forward events to Google Cloud Pub/Sub, AWS SNS/SQS, Azure Service Bus, RabbitMQ, NATS, or Kafka.

### Installation

- **Linux**: install the `sftpgo-plugins` package from the [official repository](installation.md#linux-windows-docker).
- **Windows**: plugins are included in the [SFTPGo installer](installation.md#windows).
- **Docker**: use an image tag that includes plugins. See [Docker installation](installation.md#docker).

## Plugin Development

:warning: Plugin development is an advanced topic and is not required knowledge for day-to-day usage. If you don't plan on writing any plugins, you can skip this section of the documentation.

Because SFTPGo communicates to plugins over a RPC interface, you can build and distribute a plugin for SFTPGo without having to rebuild SFTPGo itself.

Because the plugin interface is HTTP based, a plugin can in principle be developed in a different programming language. This requires re-implementing the plugin API, which is not a trivial amount of work.

Writing a plugin requires basic command-line skills and basic knowledge of the [Go programming language](https://go.dev/){:target="_blank"}.

Your plugin implementation needs to satisfy the interface for the plugin type you want to build. You can find these definitions in the [docs](https://pkg.go.dev/github.com/sftpgo/sdk/plugin#section-directories){:target="_blank"}.

The SFTPGo plugin system uses the HashiCorp [go-plugin](https://github.com/hashicorp/go-plugin){:target="_blank"} library. Please refer to its documentation for more in-depth information on writing plugins.
