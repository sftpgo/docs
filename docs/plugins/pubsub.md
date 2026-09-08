---
description: "Forward SFTPGo events to Google Cloud Pub/Sub, AWS SNS/SQS, Azure Service Bus, RabbitMQ, NATS, or Kafka for real-time integration."
---

# Pub/Sub Event Forwarding

The Pub/Sub plugin forwards SFTPGo events to external publish/subscribe messaging systems. This enables real-time integration with event-driven architectures, monitoring pipelines, and third-party services.

The plugin sends filesystem events (uploads, downloads, deletes, etc.), provider events (user/admin changes), and log events (authentication failures) as JSON messages.

## Supported services

| Service | URL scheme | Connection settings |
| ------- | ---------- | ------------------- |
| Google Cloud Pub/Sub | `gcppubsub://` | Application Default Credentials (supports Workload Identity) |
| AWS SNS | `awssns://` | Default AWS credential chain (supports IAM roles) |
| AWS SQS | `awssqs://` | Default AWS credential chain (supports IAM roles) |
| Azure Service Bus | `azuresb://` | `SERVICEBUS_CONNECTION_STRING` environment variable |
| RabbitMQ | `rabbit://` | `SFTPGO_PLUGIN_PUBSUB_RABBIT_SERVER_URL` environment variable, [TLS settings](#tls-settings) |
| NATS | `nats://` | `SFTPGO_PLUGIN_PUBSUB_NATS_SERVER_URL` environment variable, [TLS settings](#tls-settings) |
| Apache Kafka | `kafka://` | `SFTPGO_PLUGIN_PUBSUB_KAFKA_BROKERS` environment variable (comma-separated), [TLS](#tls-settings) and [SASL](#apache-kafka) settings |

## Installation

Install the `sftpgo-plugins` package as described in [Audit Logs - Installation](audit-logs.md#installation). The plugin binary is `sftpgo-plugin-pubsub`.

## Configuration

:warning: Any configuration change described below requires a service restart to take effect (e.g. `systemctl restart sftpgo`).

The topic URL is passed as the first argument to the plugin. The second argument is an optional instance identifier, useful in multi-instance deployments to distinguish which SFTPGo node generated the event.

:information_source: The examples below use plugin index `0`. If you have other plugins already configured, adjust the index accordingly. See [Plugin indexing](../plugins.md#plugin-indexing) for details.

### Environment variables

The plugin runs as a separate process and receives from SFTPGo the environment variables with the `SFTPGO_PLUGIN_PUBSUB_` prefix. All the settings documented on this page use this prefix, so they reach the plugin without additional configuration.

Variables with other names, such as `SERVICEBUS_CONNECTION_STRING` or the cloud SDK credentials, are forwarded when listed in `ENV_VARS`:

```shell
SFTPGO_PLUGINS__0__ENV_VARS="SERVICEBUS_CONNECTION_STRING"
```

Set `SFTPGO_PLUGINS__0__ENV_PREFIX="*"` to forward the whole SFTPGo environment. See [Plugins configuration](../config-file.md#plugins) for details.

The Go CDK variable names `KAFKA_BROKERS`, `NATS_SERVER_URL` and `RABBIT_SERVER_URL` remain supported when the prefixed variable is not set.

### Google Cloud Pub/Sub

```shell
SFTPGO_PLUGINS__0__TYPE=notifier
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__FS_EVENTS="upload,download,delete,rename"
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__PROVIDER_EVENTS="add,update,delete"
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__PROVIDER_OBJECTS="user,admin"
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__LOG_EVENTS="1,2"
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__RETRY_MAX_TIME=60
SFTPGO_PLUGINS__0__NOTIFIER_OPTIONS__RETRY_QUEUE_MAX_SIZE=1000
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-pubsub"
SFTPGO_PLUGINS__0__ARGS="gcppubsub://projects/my-project/topics/sftpgo-events"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

### AWS SNS

```shell
SFTPGO_PLUGINS__0__ARGS="awssns:///arn:aws:sns:us-east-2:123456789012:sftpgo-events?region=us-east-2"
```

### AWS SQS

```shell
SFTPGO_PLUGINS__0__ARGS="awssqs://sqs.us-east-2.amazonaws.com/123456789012/sftpgo-events?region=us-east-2"
```

### Azure Service Bus

```shell
SERVICEBUS_CONNECTION_STRING="Endpoint=sb://my-namespace.servicebus.windows.net/;SharedAccessKeyName=...;SharedAccessKey=..."
SFTPGO_PLUGINS__0__ENV_VARS="SERVICEBUS_CONNECTION_STRING"
SFTPGO_PLUGINS__0__ARGS="azuresb://sftpgo-events"
```

### RabbitMQ

```shell
SFTPGO_PLUGIN_PUBSUB_RABBIT_SERVER_URL="amqp://guest:guest@192.168.1.5:5672/"
SFTPGO_PLUGINS__0__ARGS="rabbit://sftpgo-events"
```

The `amqps://` scheme enables TLS. Server certificates issued by a public certificate authority are verified with the system trust store. A private certificate authority or a client certificate are configured with the [TLS settings](#tls-settings).

### NATS

```shell
SFTPGO_PLUGIN_PUBSUB_NATS_SERVER_URL="nats://192.168.1.5:4222"
SFTPGO_PLUGINS__0__ARGS="nats://sftpgo.events"
```

The `tls://` scheme enables TLS. Server certificates issued by a public certificate authority are verified with the system trust store. A private certificate authority or a client certificate are configured with the [TLS settings](#tls-settings). Credentials are set in the server URL, for example `tls://user:password@nats.example.com:4222` or `tls://token@nats.example.com:4222`.

### Apache Kafka

```shell
SFTPGO_PLUGIN_PUBSUB_KAFKA_BROKERS="192.168.1.5:9092,192.168.1.6:9092"
SFTPGO_PLUGINS__0__ARGS="kafka://sftpgo-events"
```

Kafka brokers are addressed as `host:port`, so TLS is enabled with a dedicated variable:

| Variable | Description |
| -------- | ----------- |
| `SFTPGO_PLUGIN_PUBSUB_KAFKA_TLS` | Set to `1` to connect to the brokers over TLS. TLS is also enabled when any of the [TLS settings](#tls-settings) is set |

SASL authentication is configured with the following variables:

| Variable | Description |
| -------- | ----------- |
| `SFTPGO_PLUGIN_PUBSUB_KAFKA_SASL_MECHANISM` | `PLAIN`, `SCRAM-SHA-256` or `SCRAM-SHA-512`. Setting a mechanism enables SASL |
| `SFTPGO_PLUGIN_PUBSUB_KAFKA_SASL_USERNAME` | SASL username |
| `SFTPGO_PLUGIN_PUBSUB_KAFKA_SASL_PASSWORD` | SASL password |

Example for a `SASL_SSL` listener with SCRAM authentication and a broker certificate issued by a public certificate authority:

```shell
SFTPGO_PLUGIN_PUBSUB_KAFKA_BROKERS="kafka-1.example.com:9094,kafka-2.example.com:9094"
SFTPGO_PLUGIN_PUBSUB_KAFKA_TLS=1
SFTPGO_PLUGIN_PUBSUB_KAFKA_SASL_MECHANISM="SCRAM-SHA-512"
SFTPGO_PLUGIN_PUBSUB_KAFKA_SASL_USERNAME="sftpgo"
SFTPGO_PLUGIN_PUBSUB_KAFKA_SASL_PASSWORD="secret"
SFTPGO_PLUGINS__0__ARGS="kafka://sftpgo-events"
```

:information_source: SCRAM passwords are normalized according to RFC 4013 (SASLprep), as most Kafka clients do. Passwords made of printable ASCII characters are unaffected.

### TLS settings

Kafka, RabbitMQ and NATS connections support a private certificate authority, a client certificate for mutual TLS and a server name override. The other services are always accessed over TLS with the trust store of their SDK.

| Variable | Description |
| -------- | ----------- |
| `SFTPGO_PLUGIN_PUBSUB_TLS_ROOT_CERT` | Path to a PEM-encoded CA certificate, added to the system trust store |
| `SFTPGO_PLUGIN_PUBSUB_TLS_CLIENT_CERT` | Path to a PEM-encoded client certificate |
| `SFTPGO_PLUGIN_PUBSUB_TLS_CLIENT_KEY` | Path to the private key of the client certificate. Certificate and key are set together |
| `SFTPGO_PLUGIN_PUBSUB_TLS_SERVER_NAME` | Host name expected in the server certificate. Useful when the brokers are reached by IP address, through a load balancer or a tunnel |
| `SFTPGO_PLUGIN_PUBSUB_TLS_SKIP_VERIFY` | Set to `1` to skip the server certificate verification. For diagnostics only |

Example for a Kafka cluster with a private CA and client certificate authentication:

```shell
SFTPGO_PLUGIN_PUBSUB_KAFKA_BROKERS="kafka-1.internal:9093,kafka-2.internal:9093"
SFTPGO_PLUGIN_PUBSUB_TLS_ROOT_CERT="/etc/sftpgo/kafka/ca.pem"
SFTPGO_PLUGIN_PUBSUB_TLS_CLIENT_CERT="/etc/sftpgo/kafka/client.pem"
SFTPGO_PLUGIN_PUBSUB_TLS_CLIENT_KEY="/etc/sftpgo/kafka/client.key"
SFTPGO_PLUGINS__0__ARGS="kafka://sftpgo-events"
```

The certificate files are read when the plugin starts. After renewing a certificate on disk, restart SFTPGo or wait for the plugin restart: when the connection to the service is lost the plugin exits, SFTPGo restarts it and the renewed files are loaded.

For RabbitMQ, the TLS settings require the `amqps://` scheme in the server URL. For NATS, the TLS settings enable TLS with any URL scheme.

The cloud services use the trust store of their SDK. On Linux, a private certificate authority for those services, for example for a TLS inspecting proxy, is configured with the `SSL_CERT_FILE` environment variable listed in `ENV_VARS`.

:information_source: The prefixed connection variables, the TLS settings and the SASL settings require plugin version 1.2.0 or newer.

### Adding an instance ID

To identify events from a specific SFTPGo node, add the instance ID as a second argument:

```shell
SFTPGO_PLUGINS__0__ARGS="rabbit://sftpgo-events,node-1"
```

## Event selection

Use `NOTIFIER_OPTIONS` to control which events are forwarded. See [Audit Logs - Event types](audit-logs.md#event-types) for the complete list of available events.

Configure retry behavior for transient failures:

- `RETRY_MAX_TIME` — maximum retry time in seconds for failed deliveries (default: 60)
- `RETRY_QUEUE_MAX_SIZE` — maximum number of events queued for retry (default: 1000)

## Message format

All events are published as JSON messages. Each message includes metadata attributes for routing and filtering.

### Filesystem events

Metadata: `action` (e.g., `upload`, `download`)

```json
{
  "timestamp": "2024-01-15T10:30:00.123456789Z",
  "action": "upload",
  "username": "john",
  "fs_path": "/srv/sftpgo/data/john/report.pdf",
  "virtual_path": "/report.pdf",
  "file_size": 1048576,
  "elapsed": 250,
  "status": 1,
  "protocol": "SFTP",
  "ip": "192.168.1.100",
  "session_id": "abc123",
  "fs_provider": 0,
  "role": "employee",
  "instance_id": "node-1"
}
```

| Field | Description |
| ----- | ----------- |
| `status` | `1` = OK, `2` = error, `3` = quota exceeded |
| `fs_provider` | `0` = local, `1` = S3, `2` = Google Cloud Storage, `3` = Azure Blob, `4` = encrypted local, `5` = SFTP |
| `fs_target_path`, `virtual_target_path` | Present for rename and copy operations |
| `ssh_cmd` | Present for SSH command operations |
| `open_flags` | File open flags |
| `bucket`, `endpoint` | Present for cloud storage backends |
| `role` | User role, if assigned |
| `metadata` | Custom metadata key-value pairs, if set |
| `external_username` | Username from the external authentication provider, if different from the SFTPGo username |

Fields marked with "present for" are omitted from the JSON when empty or zero.

### Provider events

Metadata: `action` (e.g., `add`), `object_type` (e.g., `user`)

```json
{
  "timestamp": "2024-01-15T10:30:00.123456789Z",
  "action": "update",
  "username": "admin",
  "ip": "192.168.1.50",
  "object_type": "user",
  "object_name": "john",
  "object_data": "eyJ1c2VyIjp7...fX0=",
  "role": "admin",
  "instance_id": "node-1"
}
```

The `object_data` field contains the object as base64-encoded JSON with sensitive fields removed. The `role` field is present only if a role is assigned.

### Log events

Metadata: `action` = `log`, `event` (integer code)

```json
{
  "timestamp": "2024-01-15T10:30:00.123456789Z",
  "event": 1,
  "protocol": "SFTP",
  "username": "unknown_user",
  "ip": "10.0.0.50",
  "message": "invalid credentials",
  "instance_id": "node-1"
}
```

The `event` field values: `1` = Login failed, `2` = Login with non-existent user, `3` = No login attempted, `4` = Algorithm negotiation failed, `5` = Login succeeded, `6` = Legal agreement accepted.
