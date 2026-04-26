---
description: "SFTP, FTP, and WebDAV gateway for any S3-compatible object storage. SFTPGo config for Cloudflare R2, Backblaze B2, AIStor, MinIO, Ceph, Wasabi and more."
---

# S3-compatible object storage services

The [S3 backend](s3.md) in SFTPGo works with any object storage service that implements the S3 API. This page documents configurations for several such services.

Each section lists the **endpoint**, any **required flags** (like path-style addressing), and known quirks. Settings not mentioned use the SFTPGo defaults.

For the full list of configuration parameters, see the [S3 backend reference](s3.md). For authentication options (access keys, IAM roles, STS, IRSA), see [Authentication](s3.md#authentication).

:information_source: There is no separate backend for Cloudflare R2, Backblaze B2, MinIO, or any other S3-compatible service — they all use the same S3 backend in SFTPGo with a custom endpoint. If your service is not listed here but advertises S3 API compatibility, the same pattern applies: set `Endpoint` to the service's S3 URL, check whether `Force path style` is required, and configure access keys as usual.

## Object Lock and checksums

If the target bucket has [Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html){:target="_blank"} (or the service's equivalent write-once/immutability feature) enabled, you must set the **Checksum algorithm** field (see the [S3 reference](s3.md#configuration)). Uploads to Object Lock-protected buckets are rejected when no checksum is present. This applies to AWS S3 and to every S3-compatible service that implements Object Lock. Not all services support every algorithm — if one fails, try another (`CRC32` is the most broadly supported).

## Cloudflare R2

[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/){:target="_blank"} is Cloudflare's S3-compatible object storage.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://<ACCOUNT_ID>.r2.cloudflarestorage.com` |
| **Region** | `auto` |
| **Force path style** | disabled (R2 supports virtual-hosted addressing) |
| **Access Key / Secret** | Create an R2 API token in the Cloudflare dashboard → R2 → Manage API Tokens |

The `<ACCOUNT_ID>` is visible in the Cloudflare dashboard under R2 → Overview. Create the bucket first via the dashboard or `wrangler r2 bucket create`.

**Notes**:

- ACL is not supported on R2 — leave this field blank. Storage class (`STANDARD` / `STANDARD_IA`) and SSE-C are supported.

## Backblaze B2

[Backblaze B2](https://www.backblaze.com/cloud-storage){:target="_blank"} exposes an S3-compatible API alongside its native B2 API.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://s3.<REGION>.backblazeb2.com` (e.g., `https://s3.us-west-004.backblazeb2.com`) |
| **Region** | The B2 region code (e.g., `us-west-004`, `eu-central-003`) |
| **Force path style** | disabled |
| **Access Key / Secret** | Create an **Application Key** in the B2 console (not the master key) |

The region code is shown next to your bucket in the B2 console. The endpoint hostname includes the same region.

**Notes**:

- Application keys can be scoped to a single bucket.

## AIStor / MinIO (self-hosted)

[AIStor](https://www.min.io/){:target="_blank"} is the S3-compatible object storage server from MinIO Inc., successor to the legacy MinIO Community Edition. Both configure identically from SFTPGo's perspective.

| Parameter | Value |
|---|---|
| **Endpoint** | Your server URL (e.g., `https://storage.internal:9000`) |
| **Region** | Any string; `us-east-1` is a common default |
| **Force path style** | **enabled** (path-style is the default; virtual-hosted requires additional domain configuration) |
| **Access Key / Secret** | Server-generated access key and secret |
| **Skip TLS verify** | Only if using self-signed certificates in development |

:information_source: The legacy **MinIO Community Edition** (`minio/minio` on GitHub) repository has been archived by MinIO Inc. Existing deployments continue to work as-is; the S3 API surface used by SFTPGo is unchanged.

## Ceph RadosGW

[Ceph Object Gateway](https://docs.ceph.com/en/latest/radosgw/){:target="_blank"} (RadosGW) is the S3-compatible front-end for Ceph clusters.

| Parameter | Value |
|---|---|
| **Endpoint** | Your RadosGW URL (e.g., `https://rgw.example.com`) |
| **Region** | The zonegroup name, or `default` if not configured |
| **Force path style** | **enabled** (recommended; virtual-hosted style requires wildcard DNS) |
| **Access Key / Secret** | RadosGW S3 user credentials (created via `radosgw-admin user create`) |

**Notes**:

- If your RadosGW is configured for virtual-hosted addressing with wildcard DNS, you can disable `Force path style`; otherwise keep it enabled.
- Server-side encryption behavior depends on your Ceph cluster's configuration.

## Wasabi

[Wasabi](https://wasabi.com/){:target="_blank"} is an S3-compatible object storage service.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://s3.<REGION>.wasabisys.com` (e.g., `https://s3.eu-central-1.wasabisys.com`) |
| **Region** | The Wasabi region (e.g., `us-east-1`, `eu-central-1`, `ap-northeast-1`) |
| **Force path style** | disabled |
| **Access Key / Secret** | Wasabi access key and secret |

See the [Wasabi service URL reference](https://knowledgebase.wasabi.com/hc/en-us/articles/360015106031){:target="_blank"} for the complete list of regional endpoints.

## DigitalOcean Spaces

[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces){:target="_blank"} is DigitalOcean's S3-compatible object storage.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://<REGION>.digitaloceanspaces.com` (e.g., `https://nyc3.digitaloceanspaces.com`) |
| **Region** | The Space region (e.g., `nyc3`, `ams3`, `sgp1`, `fra1`, `sfo3`) |
| **Force path style** | disabled |
| **Access Key / Secret** | Spaces access key and secret (create in the DigitalOcean control panel under API → Spaces access keys) |

## Hetzner Object Storage

[Hetzner Object Storage](https://www.hetzner.com/storage/object-storage/){:target="_blank"} is Hetzner Cloud's S3-compatible object storage.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://<REGION>.your-objectstorage.com` (e.g., `https://fsn1.your-objectstorage.com` for Falkenstein, `https://hel1.your-objectstorage.com` for Helsinki) |
| **Region** | The Hetzner region code (e.g., `fsn1`, `hel1`, `nbg1`) |
| **Force path style** | disabled |
| **Access Key / Secret** | Generate in the Hetzner Cloud Console under Security → Object Storage |

## Scaleway Object Storage

[Scaleway Object Storage](https://www.scaleway.com/en/object-storage/){:target="_blank"} is Scaleway's S3-compatible object storage.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://s3.<REGION>.scw.cloud` (e.g., `https://s3.fr-par.scw.cloud`, `https://s3.nl-ams.scw.cloud`) |
| **Region** | The Scaleway region (`fr-par`, `nl-ams`, `pl-waw`) |
| **Force path style** | disabled |
| **Access Key / Secret** | Create API keys in the Scaleway Console under IAM → API Keys |

## OVHcloud Object Storage

[OVHcloud Object Storage](https://www.ovhcloud.com/en/public-cloud/object-storage/){:target="_blank"} is OVHcloud's S3-compatible object storage.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://s3.<REGION>.io.cloud.ovh.net` (e.g., `https://s3.gra.io.cloud.ovh.net` for Gravelines) |
| **Region** | The OVHcloud region (`gra`, `sbg`, `bhs`, `de`, `uk`, `waw`) |
| **Force path style** | disabled |
| **Access Key / Secret** | Create S3 credentials in the OVHcloud Control Panel under Users & Roles → S3 users |

## Oracle Cloud Infrastructure (OCI) Object Storage

[OCI Object Storage](https://www.oracle.com/cloud/storage/object-storage/){:target="_blank"} exposes an S3-compatible endpoint alongside its native API.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://<NAMESPACE>.compat.objectstorage.<REGION>.oraclecloud.com` |
| **Region** | The OCI region identifier (e.g., `us-ashburn-1`, `eu-frankfurt-1`) |
| **Force path style** | **enabled** |
| **Access Key / Secret** | Create a **Customer Secret Key** in the OCI Console under Identity → Users → your user → Customer Secret Keys. OCI displays an **Access Key** alongside the **Secret Key** — use that pair as your S3 credentials. |

The `<NAMESPACE>` is your tenancy's Object Storage namespace — find it in the OCI Console under Administration → Tenancy Details, or via `oci os ns get`.

## Alibaba Cloud OSS

[Alibaba Cloud Object Storage Service (OSS)](https://www.alibabacloud.com/product/oss){:target="_blank"} provides an S3-compatible endpoint in addition to its native OSS API.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://s3.oss-<REGION>.aliyuncs.com` (e.g., `https://s3.oss-cn-hangzhou.aliyuncs.com`) |
| **Region** | The OSS region (e.g., `cn-hangzhou`, `ap-southeast-1`, `eu-central-1`) |
| **Force path style** | disabled (the S3-compatible endpoint uses virtual-hosted style) |
| **Access Key / Secret** | RAM user Access Key ID and Access Key Secret |

**Notes**:

- The S3-compatible endpoint is `s3.oss-<REGION>.aliyuncs.com` — distinct from the native OSS endpoint `oss-<REGION>.aliyuncs.com` (without the `s3.` prefix). Use the `s3.` one for SFTPGo.
- OSS has separate endpoints for internal (VPC) and public access; use the public endpoint from outside the VPC (internal form: `s3.oss-<REGION>-internal.aliyuncs.com`).
- Some OSS features (tagging, lifecycle rules) are not accessible via the S3-compatible endpoint — manage them via the OSS console or native API.

## IBM Cloud Object Storage

[IBM Cloud Object Storage](https://www.ibm.com/products/cloud-object-storage){:target="_blank"} provides an S3-compatible API on top of the COS platform.

| Parameter | Value |
|---|---|
| **Endpoint** | The regional COS endpoint (e.g., `https://s3.eu-de.cloud-object-storage.appdomain.cloud`) |
| **Region** | The COS region (`us-south`, `eu-de`, `eu-gb`, `jp-tok`, etc.) |
| **Force path style** | disabled |
| **Access Key / Secret** | Create **HMAC credentials** on the COS service credentials page (not the API key — HMAC gives you the access key / secret pair) |

The full list of regional endpoints is in the [IBM COS endpoint documentation](https://cloud.ibm.com/docs/cloud-object-storage/basics?topic=cloud-object-storage-endpoints){:target="_blank"}.

## SeaweedFS

[SeaweedFS](https://github.com/seaweedfs/seaweedfs){:target="_blank"} is an open-source distributed storage system with an S3-compatible gateway.

| Parameter | Value |
|---|---|
| **Endpoint** | Your SeaweedFS S3 gateway URL (e.g., `http://seaweedfs:8333`) |
| **Region** | Any string; `us-east-1` is safe |
| **Force path style** | **enabled** |
| **Access Key / Secret** | Credentials configured in SeaweedFS (`weed s3.configure` or the config file) |

## Storj DCS

[Storj DCS](https://www.storj.io/){:target="_blank"} is a decentralized object storage network with an S3-compatible gateway (`uplink` hosted or self-hosted).

| Parameter | Value |
|---|---|
| **Endpoint** | `https://gateway.storjshare.io` (hosted gateway) or your self-hosted gateway URL |
| **Region** | Any string (Storj does not enforce regions; `global` or `us-east-1` are common choices) |
| **Force path style** | **enabled** (recommended by Storj — the bucket name is not part of the gateway hostname) |
| **Access Key / Secret** | Create an S3 credential via the Storj satellite (`uplink access create --s3`) |

**Notes**:

- A self-hosted `uplink-s3` gateway near SFTPGo reduces latency compared to the hosted gateway.
- Storj recommends an `Upload part size` of 64 MB or higher for throughput.

## Garage

[Garage](https://garagehq.deuxfleurs.fr/){:target="_blank"} is an open-source S3-compatible object store.

| Parameter | Value |
|---|---|
| **Endpoint** | Your Garage S3 endpoint (e.g., `https://s3.garage.internal`) |
| **Region** | The region name configured in Garage (default: `garage`) |
| **Force path style** | **enabled** |
| **Access Key / Secret** | Generate via `garage key new` or the admin API |

## Tigris

[Tigris Data](https://www.tigrisdata.com/){:target="_blank"} is an S3-compatible storage service.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://t3.storage.dev` for general clients; `https://fly.storage.tigris.dev` when SFTPGo runs inside Fly.io |
| **Region** | `auto` |
| **Force path style** | disabled |
| **Access Key / Secret** | Generate in the Tigris console or via `fly storage create` on Fly.io |

## Supabase Storage

[Supabase Storage](https://supabase.com/storage){:target="_blank"} exposes an S3-compatible endpoint.

| Parameter | Value |
|---|---|
| **Endpoint** | `https://<PROJECT_REF>.storage.supabase.co/storage/v1/s3` |
| **Region** | The region selected when creating the Supabase project |
| **Force path style** | **enabled** |
| **Access Key / Secret** | Create S3 credentials in the Supabase dashboard under Project Settings → Storage → S3 Access Keys |

## Troubleshooting checklist

If a newly configured S3-compatible service is not working:

1. **Connectivity**: can the SFTPGo host reach the endpoint? Test with `curl -v <endpoint>` — expect a response from the service, not a network error.
2. **Path vs virtual-hosted style**: try toggling `Force path style`. Path-style (`endpoint/bucket/key`) is safer for most third-party services; virtual-hosted (`bucket.endpoint/key`) requires the service to resolve the bucket subdomain.
3. **Region**: some services (MinIO, SeaweedFS, Garage) accept any region string, others (AWS, Wasabi, DigitalOcean Spaces) enforce exact regional matches.
4. **TLS**: self-signed certificates fail unless `Skip TLS verify` is enabled (testing only). For production, use a publicly trusted certificate or add the CA to the system trust store.
5. **Credentials scope**: make sure the access key has permissions on the specific bucket, not just "read" when you need "write". Many services distinguish read-only, write-only, and full-access keys.

For protocol-level debugging, enable SFTPGo's debug logs: `log_level: "debug"` in the configuration — S3 SDK errors and HTTP response codes will appear in the log output.

## Service not listed?

If your S3-compatible service works but isn't documented here, please [open an issue](https://github.com/drakkan/sftpgo/issues){:target="_blank"} with the configuration that worked — we will add it to this page. Any service that advertises S3 API compatibility is expected to work with the existing S3 backend; the only code-level adjustments SFTPGo has made for non-AWS services are around optional features (request checksums, storage class mappings), all of which are configurable.
