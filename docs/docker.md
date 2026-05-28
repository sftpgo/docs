---
description: "Run SFTPGo Enterprise with Docker or Kubernetes. Pull from the official registry, configure with environment variables, deploy with Docker Compose or Helm chart."
---

# Docker

SFTPGo Enterprise is accessible through our Docker repository: `registry.sftpgo.com/sftpgo`

## Latest tags

- registry.sftpgo.com/sftpgo/sftpgo:v2.7.20260528
- registry.sftpgo.com/sftpgo/sftpgo:v2.7.20260528-plugins
- registry.sftpgo.com/sftpgo/sftpgo:v2.7.20260528-distroless
- registry.sftpgo.com/sftpgo/sftpgo:v2.7.20260528-distroless-plugins

## How to use the SFTPGo image

### Start a `sftpgo` server instance

Starting a SFTPGo instance is simple:

```shell
docker run --name some-sftpgo -p 8080:8080 -p 2022:2022 -d "registry.sftpgo.com/sftpgo/sftpgo:<tag>"
```

... where `some-sftpgo` is the name you want to assign to your container, and `tag` is the tag specifying the SFTPGo version you want. See the list above for relevant tags.

Now visit [http://localhost:8080/web/admin](http://localhost:8080/web/admin){:target="_blank"}, replacing `localhost` with the appropriate IP address if SFTPGo is not reachable on localhost, create the first admin and a new SFTPGo user. The SFTP service is available on port 2022.

If you don't want to persist any files, for example for testing purposes, you can run an SFTPGo instance like this:

```shell
docker run --rm --name some-sftpgo -p 8080:8080 -p 2022:2022 -d "registry.sftpgo.com/sftpgo/sftpgo:<tag>"
```

### Container shell access and viewing SFTPGo logs

The docker exec command allows you to run commands inside a Docker container. The following command line will give you a shell inside your `sftpgo` container:

```shell
docker exec -it some-sftpgo sh
```

The logs are available through Docker's container log:

```shell
docker logs some-sftpgo
```

### Container graceful shutdown

```shell
docker run --name some-sftpgo \
    -p 2022:2022 \
    -e SFTPGO_GRACE_TIME=32 \
    -d "registry.sftpgo.com/sftpgo/sftpgo:<tag>"
```

Setting the `SFTPGO_GRACE_TIME` environment variable to a non zero value when creating or running a container will enable a graceful shutdown period in seconds that will allow existing connections to hopefully complete before being forcibly closed when the time has passed.

While the SFTPGo container is in graceful shutdown mode waiting for the last connection(s) to finish, no new connections will be allowed.

If no connections are active or `SFTPGO_GRACE_TIME=0` (default value if unset) the container will shutdown immediately.

:warning: The default docker grace time is 10 seconds, so if your SFTPGO_GRACE_TIME is larger than the docker grace time, then any `docker stop some-sftpgo` command will terminate your container once the docker grace time has passed. To ensure that the full SFTPGO_GRACE_TIME can be used, you can send a SIGINT or SIGTERM signal. Those signals can be sent using one of these commands: `docker kill --signal=SIGINT some-sftpgo` or `docker kill --signal=SIGTERM some-sftpgo`.
Alternatively you can increase the default docker grace time to a value larger than your SFTPGO_GRACE_TIME. The default docker grace time can either be specified at creation/run time using `--stop-timeout <value>` or you can simply add `--time <value>` to the docker stop command like in this 60 seconds example `docker stop --time 60 some-sftpgo`.

### Where to Store Data

Important note: There are several ways to store data used by applications that run in Docker containers. We encourage users of the SFTPGo images to familiarize themselves with the options available, including:

- Let Docker manage the storage for SFTPGo data by [writing them to disk on the host system using its own internal volume management](https://docs.docker.com/engine/tutorials/dockervolumes/#adding-a-data-volume){:target="_blank"}. This is the default and is easy and fairly transparent to the user. The downside is that the files may be hard to locate for tools and applications that run directly on the host system, i.e. outside containers.
- Create a data directory on the host system (outside the container) and [mount this to a directory visible from inside the container](https://docs.docker.com/engine/tutorials/dockervolumes/#mount-a-host-directory-as-a-data-volume){:target="_blank"}. This places the SFTPGo files in a known location on the host system, and makes it easy for tools and applications on the host system to access the files. The downside is that the user needs to make sure that the directory exists, and that e.g. directory permissions and other security mechanisms on the host system are set up correctly. The SFTPGo image runs using `1000` as UID/GID by default.

The Docker documentation is a good starting point for understanding the different storage options and variations, and there are multiple blogs and forum postings that discuss and give advice in this area. We will simply show the basic procedure here for the latter option above:

1. Create a data directory on a suitable volume on your host system, e.g. `/my/own/sftpgodata`. The user with ID `1000` must be able to write to this directory. Please note that you don't need an actual user with ID `1000` on your host system: `chown -R 1000:1000 /my/own/sftpgodata` is enough even if there is no user/group with UID/GID `1000`.
2. Create a home directory for the sftpgo container user on your host system e.g. `/my/own/sftpgohome`. As with the data directory above, make sure that the user with ID `1000` can write to this directory: `chown -R 1000:1000 /my/own/sftpgohome`
3. Start your SFTPGo container like this:

```shell
docker run --name some-sftpgo \
    -p 8080:8080 \
    -p 2022:2022 \
    --mount type=bind,source=/my/own/sftpgodata,target=/srv/sftpgo \
    --mount type=bind,source=/my/own/sftpgohome,target=/var/lib/sftpgo \
    -d "registry.sftpgo.com/sftpgo/sftpgo:<tag>"
```

As you can see SFTPGo uses two main volumes:

- `/srv/sftpgo` to handle persistent data. The default home directory for SFTP/FTP/WebDAV users is `/srv/sftpgo/data/<username>`. Backups are stored in `/srv/sftpgo/backups`
- `/var/lib/sftpgo` is the home directory for the sftpgo system user defined inside the container. This is the container working directory too, host keys will be created here when using the default configuration.

If you want to get fine grained control, you can also mount `/srv/sftpgo/data` and `/srv/sftpgo/backups` as separate volumes instead of mounting `/srv/sftpgo`.

### Configuration

The runtime configuration can be customized via environment variables that you can set passing the `-e` option to the `docker run` command or inside the `environment` section if you are using [docker stack deploy](https://docs.docker.com/engine/reference/commandline/stack_deploy/){:target="_blank"} or [docker-compose](https://github.com/docker/compose){:target="_blank"}.

Refer to the [documentation](env-vars.md) to learn how to configure SFTPGo using environment variables.

Alternately you can mount your custom configuration file to `/var/lib/sftpgo` or `/var/lib/sftpgo/.config/sftpgo`.

### Running as an arbitrary user

The SFTPGo image runs using `1000` as UID/GID by default. If you know the permissions of your data and/or configuration directory are already set appropriately or you have need of running SFTPGo with a specific UID/GID, it is possible to invoke this image with `--user` set to any value (other than `root/0`) in order to achieve the desired access/configuration:

```shell
$ ls -lnd data
drwxr-xr-x 2 1100 1100 6  7 nov 09.09 data
$ ls -lnd config
drwxr-xr-x 2 1100 1100 6  7 nov 09.19 config
```

With the above directory permissions, you can start a SFTPGo instance like this:

```shell
docker run --name some-sftpgo \
    --user 1100:1100 \
    -p 8080:8080 \
    -p 2022:2022 \
    --mount type=bind,source="${PWD}/data",target=/srv/sftpgo \
    --mount type=bind,source="${PWD}/config",target=/var/lib/sftpgo \
    -d "registry.sftpgo.com/sftpgo/sftpgo:<tag>"
```

Alternately build your own image using the official one as a base, here is a sample Dockerfile:

```shell
FROM registry.sftpgo.com/sftpgo/sftpgo:<tag>
USER root
RUN chown -R 1100:1100 /etc/sftpgo && chown 1100:1100 /var/lib/sftpgo /srv/sftpgo
USER 1100:1100
```

## Image Variants

The `sftpgo` images comes in many flavors, each designed for a specific use case.

### `sftpgo:<version>`

This is the defacto image, it is based on [Debian](https://www.debian.org/){:target="_blank"}, available in [the `debian` official image](https://hub.docker.com/_/debian){:target="_blank"}. If you are unsure about what your needs are, you probably want to use this one.

### `sftpgo:<version>-plugins`

These tags provide the standard image with the addition of plugins installed in `/usr/local/bin`.

### `sftpgo:<version>-distroless`

This image is based on the popular [Distroless project](https://github.com/GoogleContainerTools/distroless){:target="_blank"}. We use the latest Debian based distroless image as base.

Distroless variant contains only a statically linked sftpgo binary and its minimal runtime dependencies and so it doesn't allow shell access (no shell is installed).
SQLite support is disabled since it requires CGO and so a C runtime which is not installed.
The default data provider is `bolt`, all the supported data providers except `sqlite` work.

:bulb: When running cloud-backend users (S3, GCS, Azure Blob) on distroless or other minimal containers with a read-only or ephemeral filesystem, enable in-memory transfer pipes so no writable temp directory is needed:

```shell
SFTPGO_HOOK__MEMORY_PIPES__ENABLED=1
```

Without this, uploads against cloud users may fail when SFTPGo cannot create a file-backed pipe. This is the default in SFTPGo cloud-marketplace offerings.

### `sftpgo:<version>-distroless-plugins`

These tags provide the distroless image with the addition of open source and proprietary plugins installed in `/usr/local/bin`.

## Kubernetes

SFTPGo can be deployed on Kubernetes using the official [Helm chart](https://github.com/sftpgo/helm-chart){:target="_blank"}.

### Installing the chart

```shell
helm install sftpgo oci://ghcr.io/sftpgo/helm-charts/sftpgo
```

### Configuration

The chart is configured through `values.yaml`. You can override values at install time:

```shell
helm install sftpgo oci://ghcr.io/sftpgo/helm-charts/sftpgo -f my-values.yaml
```

Key configuration options:

```yaml
# Number of replicas for high availability
replicaCount: 2

# Use the Enterprise image with plugins
image:
  repository: registry.sftpgo.com/sftpgo/sftpgo
  tag: v2.7.20260528-plugins

# Enable protocols
sftpd:
  enabled: true
  port: 2022

ftpd:
  enabled: false
  port: 2021

webdavd:
  enabled: false
  port: 8081

httpd:
  enabled: true
  port: 8080

# Pass SFTPGo configuration as environment variables
env:
  SFTPGO_DATA_PROVIDER__DRIVER: postgresql
  SFTPGO_DATA_PROVIDER__NAME: sftpgo
  SFTPGO_DATA_PROVIDER__HOST: postgres-host
  SFTPGO_DATA_PROVIDER__PORT: "5432"

# Or reference Kubernetes secrets for sensitive values
envFrom:
  - secretRef:
      name: sftpgo-db-credentials
```

### Exposing SFTP externally

SFTP, FTP, and WebDAV are plain TCP protocols (not HTTP), so they cannot be exposed through a standard HTTP / Layer 7 Ingress controller. Use a Kubernetes **Service** of type `LoadBalancer` (on cloud providers) or `NodePort` for external access. The HTTP-based WebAdmin and WebClient, on the other hand, can be fronted by an Ingress.

A typical shape:

```yaml
service:
  sftpd:
    type: LoadBalancer
    port: 22            # external port
    annotations:
      # Cloud-specific annotations go here, for example:
      # service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      # service.beta.kubernetes.io/azure-load-balancer-resource-group: "mygroup"

  httpd:
    type: ClusterIP     # exposed via Ingress below

ingress:
  httpd:
    enabled: true
    className: nginx
    hosts:
      - host: sftpgo.example.com
        paths: ["/"]
    tls:
      - secretName: sftpgo-tls
        hosts: [sftpgo.example.com]
```

An NGINX Ingress controller with the [TCP services ConfigMap](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/){:target="_blank"} can forward a low-level TCP port to the SFTP service if a dedicated LoadBalancer is not an option, but a cloud LoadBalancer (NLB on AWS, TCP LB on Azure / GCP) is the usual choice.

### Persistence

On a local filesystem backend, users' home directories must survive pod restarts. Mount a `PersistentVolumeClaim` for `/srv/sftpgo`:

```yaml
persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 100Gi
  storageClassName: standard
```

If you run **cloud-backend users only** (S3 / GCS / Azure Blob, with no local user home directories), the home PVC is optional — enable [in-memory transfer pipes](env-vars.md) instead so SFTPGo does not need writable local scratch space:

```yaml
env:
  SFTPGO_HOOK__MEMORY_PIPES__ENABLED: "1"
```

For multi-replica deployments that share the same local files, you need a `ReadWriteMany` storage class (e.g. EFS on AWS, Azure Files, Filestore on GCP). For cloud-backend-only deployments there is no shared state on the filesystem, so `ReadWriteOnce` per-pod ephemeral volumes are enough.

### Health probes and metrics

The chart wires up liveness and readiness probes automatically against the built-in telemetry server, which is **always enabled on port 10000** inside the pod (the chart sets `SFTPGO_TELEMETRY__BIND_PORT=10000` and exposes a named port `telemetry`). You do not need to configure probes by hand — they point at `/healthz` on that port.

The same port serves Prometheus metrics at `/metrics` (see [Metrics](metrics.md)) and the profiler at `/debug/pprof` (when `enable_profiler` is set — see [Profiling](profiling.md)). If you run the Prometheus Operator, enable the bundled `PodMonitor`:

```yaml
podMonitor:
  enabled: true
```

### License key and secrets

Ship the license key and database password through a Kubernetes `Secret`:

```yaml
envFrom:
  - secretRef:
      name: sftpgo-secrets

# Secret (created separately):
# apiVersion: v1
# kind: Secret
# metadata:
#   name: sftpgo-secrets
# type: Opaque
# stringData:
#   SFTPGO_LICENSE_KEY: XXXX-XXXX-XXXX-XXXX
#   SFTPGO_DATA_PROVIDER__PASSWORD: your-db-password
```

### Host keys across replicas and restarts

SFTPGo generates per-instance SSH host keys the first time it starts if none are configured. In Kubernetes (and in any ephemeral / multi-replica deployment) that default is problematic: each replica ends up with its own keys, and clients see a key-mismatch warning as they hit different pods.

There are two clean ways to share the same host keys across pods:

1. **Store host keys in the data provider (recommended).** In the WebAdmin, go to **Server Manager → Configurations → SFTP** and paste the private keys directly; SFTPGo stores them encrypted in the database and every replica that connects to the shared data provider loads the same set. This approach also works on Linux and Windows installs — it removes any need to synchronize key files on the filesystem, and the keys survive container rebuilds.
2. **Mount keys as a Kubernetes Secret and reference them from the config.** Classic pattern, useful if you prefer keeping keys outside the database or you're managing them with an external tool (sealed-secrets, external-secrets, cert-manager for host certificates, etc.):

    ```yaml
    # Create the secret once:
    # kubectl create secret generic sftpgo-host-keys \
    #   --from-file=id_rsa=./id_rsa \
    #   --from-file=id_ecdsa=./id_ecdsa \
    #   --from-file=id_ed25519=./id_ed25519

    extraVolumes:
      - name: host-keys
        secret:
          secretName: sftpgo-host-keys
          defaultMode: 0400

    extraVolumeMounts:
      - name: host-keys
        mountPath: /etc/sftpgo/host-keys
        readOnly: true

    env:
      SFTPGO_SFTPD__HOST_KEYS: "/etc/sftpgo/host-keys/id_rsa,/etc/sftpgo/host-keys/id_ecdsa,/etc/sftpgo/host-keys/id_ed25519"
    ```

    (Field names follow the Helm chart conventions; check the chart documentation for the exact names.)

Rotate by updating the Secret (or the WebAdmin entry) and reloading SFTPGo.

### Data provider

SQLite works for single-replica deployments but is not suitable for multi-replica: with more than one replica, point SFTPGo at an external PostgreSQL, MySQL, or CockroachDB instance (managed DB services or an in-cluster database operator). See [Data provider management](data-provider.md) for the migration / concurrency considerations.

For the full list of configurable values, see the chart [documentation](https://github.com/sftpgo/helm-chart){:target="_blank"}.

### Marketplace container offerings

SFTPGo is also available as a managed container offering on cloud marketplaces:

- **AWS**: [SFTPGo Enterprise - Container](https://aws.amazon.com/marketplace/pp/prodview-y2aqp5bjdjdvu){:target="_blank"} for Amazon ECS and EKS.
- **Azure**: [SFTPGo Enterprise for AKS](https://marketplace.microsoft.com/product/container/eliamarzia1667381463185.sftpgo_enterprise_aks){:target="_blank"} for Azure Kubernetes Service.
