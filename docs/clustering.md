---
description: "SFTPGo clustering and high availability: multiple nodes behind a load balancer with a shared PostgreSQL or MySQL database, on VMs or Kubernetes."
---

# Clustering and high availability

Several SFTPGo nodes can serve the same users behind a load balancer. Adding nodes increases the number of connections and transfers the service can handle, and when a node stops, for an upgrade or a failure, the load balancer sends new connections to the others. A *node* is a single running SFTPGo instance; on Kubernetes, a pod.

The default configuration is intended for a single instance: the sections below describe the settings a cluster requires.

## How it works

The nodes share one data provider. Users, groups, virtual folders, event rules and the settings saved from **Server Manager > Configurations** are stored once and served by every node. Disk and data-transfer quotas are enforced across the nodes, and scheduled event rules run on one node at a time.

With the HA feature, part of the [Ultimate tier](https://sftpgo.com/on-premises){:target="_blank"}, the nodes also communicate with each other through the REST API and work as a single service:

- **Changes apply immediately.** A change to event rules, configurations, IP lists or the license, saved on any node, is applied by every node as soon as it is saved.
- **Sessions are shared.** WebAdmin and WebClient sessions, REST API tokens and public share cookies issued by one node are accepted by all of them and remain valid when a node restarts. The load balancer can send each HTTP request to any node.
- **Connections are managed cluster-wide.** WebAdmin and the REST API list and close the connections of all the nodes. Deleting a user, or updating it with the option to disconnect it, closes its connections on every node.

Connection, transfer and rate limits apply per node: see [node](config-file.md#node).

## Prerequisites

- A database reachable from every node: PostgreSQL (or compatible, such as CockroachDB) or MySQL/MariaDB (or compatible).
- For users that store files on the local filesystem, the same files available to every node at the same path, for example on shared or replicated storage. Cloud storage backends (S3, Azure Blob Storage, Google Cloud Storage) require no additional setup.

## Configuration

The examples use [environment variables](env-vars.md). Changes to the configuration file or to environment variables take effect after a restart of the service, e.g. `systemctl restart sftpgo`. The [node](#node) and [signing passphrase](#signing-passphrase) settings take effect with the HA feature.

### Data provider

Point every node at the same database and declare it shared. For example, a PostgreSQL database named `sftpgo` on `192.168.1.5`:

```shell
SFTPGO_DATA_PROVIDER__DRIVER=postgresql
SFTPGO_DATA_PROVIDER__NAME=sftpgo
SFTPGO_DATA_PROVIDER__HOST="192.168.1.5"
SFTPGO_DATA_PROVIDER__PORT=5432
SFTPGO_DATA_PROVIDER__USERNAME=username
SFTPGO_DATA_PROVIDER__PASSWORD=mysecret
SFTPGO_DATA_PROVIDER__IS_SHARED=1
```

For MySQL/MariaDB, set the driver to `mysql` and adjust the other values.

### Node

Each node declares the address the other nodes use to reach its REST API: its own IP address or host name, and the port and protocol of an HTTP binding that serves the REST API. The nodes must be able to connect to each other on that port.

```shell
# Different on each node
SFTPGO_DATA_PROVIDER__NODE__HOST="192.168.1.2"
# Port of an HTTP binding with the REST API enabled, 8080 by default
SFTPGO_DATA_PROVIDER__NODE__PORT=8080
# http or https, matching the binding
SFTPGO_DATA_PROVIDER__NODE__PROTO=http
```

On Kubernetes, use the pod IP:

```yaml
- name: SFTPGO_DATA_PROVIDER__NODE__HOST
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
```

### Signing passphrase

Set the same passphrase on every node, so that sessions and tokens issued by one node are accepted by the others. Use a long random string.

```shell
SFTPGO_HTTPD__SIGNING_PASSPHRASE='a long random string'
```

### Host keys

Every node must present the same SSH host keys: otherwise SFTP clients warn that the host key has changed when the load balancer sends them to a different node than the first time.

Configure the host keys from **Server Manager > Configurations > SFTP**: they are stored, encrypted, in the data provider and used by every node, and they take precedence over the key files.

Alternatively, copy the key files of one node to all the others: `id_rsa`, `id_ecdsa`, `id_ed25519` and their `.pub` files, in the configuration directory (`/etc/sftpgo` for DEB and RPM installations). On Docker and Kubernetes, mount the keys as secrets and list their paths:

```yaml
- name: SFTPGO_SFTPD__HOST_KEYS
  value: "/etc/sftpgo/ssh-keys/id_rsa,/etc/sftpgo/ssh-keys/id_ecdsa"
```

### Master key

If you set a [KMS](kms.md) master key, set the same one on every node: each node decrypts the secrets stored by the others.

### Client IP addresses

Behind a load balancer, SFTPGo sees the address of the load balancer unless the original client address is forwarded. Load balancers that preserve the client address need no configuration. Otherwise:

- for SFTP and FTP, enable the PROXY protocol, versions 1 and 2, on both the load balancer and SFTPGo:

    ```shell
    # 1: the PROXY header is optional, 2: it is required
    SFTPGO_COMMON__PROXY_PROTOCOL=2
    # Addresses of the load balancer, allowed to send the PROXY header
    SFTPGO_COMMON__PROXY_ALLOWED="192.168.1.1/32"
    # Addresses allowed to connect without the PROXY header
    SFTPGO_COMMON__PROXY_SKIPPED="192.168.0.0/24"
    ```

- for the web interfaces and the REST API, read the address from the header set by the load balancer, or use the PROXY protocol on the binding:

    ```shell
    SFTPGO_HTTPD__BINDINGS__0__PROXY_ALLOWED="192.168.1.1/32"
    SFTPGO_HTTPD__BINDINGS__0__CLIENT_IP_PROXY_HEADER="X-Forwarded-For"
    ```

The client address is what [brute force protection](defender.md), [rate limiting](rate-limiting.md), IP lists and the audit log work on. See [Reverse proxies and load balancers](web-interfaces.md#reverse-proxies-and-load-balancers) for the other HTTP settings and [FTP](ftp.md) for the data connections.

### Other recommendations

- Set the [defender](defender.md) `driver` to `provider`, so that a ban applies to every node.
- With cloud storage backends, enable in-memory pipes: transfers then stream through memory instead of temporary files on the local disk, which would otherwise need as much free space as the files being transferred.

    ```shell
    SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1
    ```

## Load balancer

- **SFTP and FTP** are TCP protocols: balance them at the TCP level.
- **HTTP**: every node serves any request. Chunked WebClient uploads ([TUS](web-interfaces.md#reverse-proxies-and-load-balancers)) require session affinity.
- **Health checks**: every HTTP binding answers `GET /healthz`, as does the [telemetry](metrics.md) server.
