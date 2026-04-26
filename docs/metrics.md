---
description: "Monitor SFTPGo with Prometheus metrics and the official Grafana dashboard. Track transfers, authentication, per-user activity, and backend operations."
---

# Metrics

SFTPGo exposes [Prometheus](https://prometheus.io/){:target="_blank"}-compatible metrics via the `/metrics` endpoint on the telemetry server.

## Enabling metrics

The telemetry server is disabled by default. To enable it, set the following environment variables:

```shell
SFTPGO_TELEMETRY__BIND_PORT=10000
SFTPGO_TELEMETRY__BIND_ADDRESS=0.0.0.0
```

Use `0.0.0.0` to listen on all interfaces, or `127.0.0.1` if Prometheus runs on the same host. Port `10000` is the convention used throughout this documentation and is the fixed port the official Helm chart binds to.

Metrics are then available at `http://<host>:10000/metrics`.

The telemetry server also exposes:

- `/healthz` — health check endpoint (always unauthenticated).
- `/debug/pprof` — profiling data, if `enable_profiler` is set to `true`. See [Profiling](profiling.md).

### Securing the endpoint

You can protect the telemetry endpoint with HTTP basic authentication by setting `auth_user_file` to a path to an htpasswd-format file (bcrypt or md5 crypt hashes). The `/healthz` endpoint is always unauthenticated.

```shell
SFTPGO_TELEMETRY__AUTH_USER_FILE=/etc/sftpgo/telemetry.htpasswd
```

Create the htpasswd file:

```text
htpasswd -Bc /etc/sftpgo/telemetry.htpasswd metrics_user
```

For HTTPS:

```shell
SFTPGO_TELEMETRY__CERTIFICATE_FILE=/path/to/cert.pem
SFTPGO_TELEMETRY__CERTIFICATE_KEY_FILE=/path/to/key.pem
```

## Available metrics

### Service health

- `sftpgo_dataprovider_availability` — Whether the data provider is reachable (gauge: 1 = available, 0 = unavailable).
- `sftpgo_active_connections` — Number of currently active connections (gauge).

### File transfers

Global counters aggregated across all protocols and backends.

- `sftpgo_uploads_total` — Total number of successful uploads.
- `sftpgo_upload_errors_total` — Total failed uploads.
- `sftpgo_downloads_total` — Total number of successful downloads.
- `sftpgo_download_errors_total` — Total failed downloads.
- `sftpgo_upload_size_bytes` — Total bytes uploaded (including partial uploads).
- `sftpgo_download_size_bytes` — Total bytes downloaded (including partial downloads).

### Filesystem operations

Protocol-agnostic counters for filesystem operations. These include operations initiated by users, event rules, and data retention checks.

- `sftpgo_fs_ops_total{operation="..."}` — Total filesystem operations by type.
- `sftpgo_fs_ops_errors_total{operation="..."}` — Total filesystem operation errors by type.

Operation values: `rename`, `delete`, `rmdir`, `copy`, `mkdir`.

### Authentication

- `sftpgo_login_total{method="...", result="..."}` — Login events by authentication method and result.
- `sftpgo_no_auth_total` — Clients disconnected for inactivity before attempting authentication.

Method values: `password`, `publickey`, `keyboard-interactive`, `publickey+password`, `publickey+keyboard-interactive`, `TLSCertificate`, `TLSCertificate+password`, `IDP`.

Result values: `attempt`, `ok`, `ko`.

### Storage backend transfers

Per-backend breakdown of transfer activity. Available backends: `s3`, `gcs`, `azblob`, `sftpfs`, `ftpfs`, `httpfs`.

- `sftpgo_backend_uploads_total{backend="..."}` — Backend uploads.
- `sftpgo_backend_upload_errors_total{backend="..."}` — Backend upload errors.
- `sftpgo_backend_upload_size_bytes{backend="..."}` — Backend upload bytes.
- `sftpgo_backend_downloads_total{backend="..."}` — Backend downloads.
- `sftpgo_backend_download_errors_total{backend="..."}` — Backend download errors.
- `sftpgo_backend_download_size_bytes{backend="..."}` — Backend download bytes.

### Storage backend operations

Metadata operation counters for cloud backends only (`s3`, `gcs`, `azblob`). These track individual API calls relevant for cost monitoring and throttling analysis.

- `sftpgo_backend_ops_total{backend="...", operation="..."}` — Backend metadata operations.
- `sftpgo_backend_ops_errors_total{backend="...", operation="..."}` — Backend metadata operation errors.

Operation values: `list`, `copy`, `delete`, `head`.

:information_source: Filesystem operations (`sftpgo_fs_ops_total`) and backend operations (`sftpgo_backend_ops_total`) track different things. Filesystem operations count logical user actions (e.g., one directory rename). Backend operations count actual cloud API calls (e.g., the same rename on S3 generates N copy + N delete API calls, one per file in the directory).

### Per-user metrics

Per-user metrics provide visibility into individual user activity. These are always enabled when the telemetry server is active.

- `sftpgo_user_active_connections{username="..."}` — Active sessions per user (gauge).
- `sftpgo_user_uploads_total{username="..."}` — Successful uploads per user.
- `sftpgo_user_upload_errors_total{username="..."}` — Upload errors per user.
- `sftpgo_user_downloads_total{username="..."}` — Successful downloads per user.
- `sftpgo_user_download_errors_total{username="..."}` — Download errors per user.
- `sftpgo_user_upload_size_bytes{username="..."}` — Upload bytes per user.
- `sftpgo_user_download_size_bytes{username="..."}` — Download bytes per user.
- `sftpgo_user_login_ok_total{username="..."}` — Successful logins per user.
- `sftpgo_user_login_ko_total{username="..."}` — Failed logins per user.

The `__system__` username appears for operations executed by the event manager (scheduled rules, ICAP actions, etc.). To exclude it from queries: `{username!="__system__"}`.

Stale user entries (no active connections and no activity for 24 hours) are automatically cleaned up when the tracked user count exceeds 2000.

### Process metrics

Standard Go process metrics are also exported:

- Memory usage (`process_resident_memory_bytes`, `go_memstats_heap_inuse_bytes`).
- CPU usage (`process_cpu_seconds_total`).
- Open file descriptors (`process_open_fds`, `process_max_fds`).
- Internal threads (`go_goroutines`).
- Process start time (`process_start_time_seconds`).

## Prometheus configuration

Add SFTPGo as a scrape target in your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: sftpgo
    scrape_interval: 15s
    static_configs:
      - targets: ["<sftpgo-host>:10000"]
```

If basic auth is enabled:

```yaml
scrape_configs:
  - job_name: sftpgo
    scrape_interval: 15s
    basic_auth:
      username: metrics_user
      password: your_password
    static_configs:
      - targets: ["<sftpgo-host>:10000"]
```

## Grafana dashboard

A ready-to-use [Grafana dashboard for SFTPGo Enterprise](https://grafana.com/grafana/dashboards/25177-sftpgo-enterprise/){:target="_blank"} is available on Grafana Labs. Import it by ID `25177` or via the dashboard JSON to visualize transfer activity, authentication events, per-user metrics, storage backend operations, and system health out of the box.

## Monitoring platform compatibility

SFTPGo metrics are compatible with any monitoring platform that supports Prometheus scraping. No SFTPGo-side changes are needed — the integration is configured on the agent/collector side.

- **Grafana / Grafana Cloud** — native Prometheus scraping via Alloy or Grafana Agent.
- **Datadog** — Datadog Agent with [OpenMetrics check](https://docs.datadoghq.com/integrations/openmetrics/){:target="_blank"}.
- **New Relic** — `nri-prometheus` or Prometheus remote write integration.
- **Elastic** — Metricbeat with Prometheus module.
- **Splunk** — Splunk OTel Collector with Prometheus receiver.
- **AWS CloudWatch** — CloudWatch Agent with Prometheus scrape configuration.
- **OpenTelemetry Collector** — `prometheus` receiver with any OTLP exporter.

For structured log ingestion, SFTPGo outputs JSON logs. Any log collector with a JSON parser (Promtail, Filebeat, Datadog Agent, Fluentd) can ingest them natively.
