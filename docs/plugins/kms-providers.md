---
description: "Encrypt SFTPGo secrets with cloud KMS: Google Cloud KMS, AWS KMS, Azure Key Vault, HashiCorp Vault, or Oracle Key Vault. Managed identity supported."
---

# Cloud KMS Providers

The KMS plugin adds support for cloud-based Key Management Services to SFTPGo. For general information about how SFTPGo handles secrets, see [Key Management Services](../kms.md).

## Supported providers

| Provider | URL scheme | Managed identity support |
| -------- | ---------- | ----------------------- |
| Google Cloud KMS | `gcpkms://` | Yes (Application Default Credentials) |
| AWS KMS | `awskms://` | Yes (IAM roles, instance profiles) |
| Azure Key Vault | `azurekeyvault://` | Yes (Azure Managed Identity) |
| HashiCorp Vault | `hashivault://` | N/A |
| Oracle Key Vault | `ocikeyvault://` | Yes (Instance Principal) |

## Installation

Install the `sftpgo-plugins` package as described in [Audit Logs - Installation](audit-logs.md#installation). The plugin binary is `sftpgo-plugin-kms`.

## Configuration

:warning: Any configuration change described below requires a service restart to take effect (e.g. `systemctl restart sftpgo`).

The KMS plugin is configured in the SFTPGo main configuration file or via environment variables. You need to configure:

1. The `kms.secrets.url` setting with the KMS key URL.
2. A plugin entry pointing to the plugin binary.

:information_source: The examples below use plugin index `0`. If you have other plugins already configured, adjust the index accordingly. See [Plugin indexing](../plugins.md#plugin-indexing) for details.

### Google Cloud KMS

```shell
SFTPGO_KMS__SECRETS__URL="gcpkms://projects/my-project/locations/us-east1/keyRings/my-keyring/cryptoKeys/my-key"

SFTPGO_PLUGINS__0__TYPE=kms
SFTPGO_PLUGINS__0__KMS_OPTIONS__SCHEME=gcpkms
SFTPGO_PLUGINS__0__KMS_OPTIONS__ENCRYPTED_STATUS=GCP
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-kms"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

Authentication uses [Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials){:target="_blank"}, which automatically supports Workload Identity on GKE, attached service accounts on Compute Engine, and `GOOGLE_APPLICATION_CREDENTIALS` environment variable.

### AWS KMS

```shell
# Using key ID
SFTPGO_KMS__SECRETS__URL="awskms://1234abcd-12ab-34cd-56ef-1234567890ab?region=us-east-1"
# Or using alias
SFTPGO_KMS__SECRETS__URL="awskms://alias/my-key-alias?region=us-east-1"
# Or using ARN
SFTPGO_KMS__SECRETS__URL="arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab"

SFTPGO_PLUGINS__0__TYPE=kms
SFTPGO_PLUGINS__0__KMS_OPTIONS__SCHEME=awskms
SFTPGO_PLUGINS__0__KMS_OPTIONS__ENCRYPTED_STATUS=AWS
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-kms"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

Authentication uses the default AWS credential chain, which automatically supports IAM roles for EC2, ECS task roles, and standard environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).

### Azure Key Vault

```shell
SFTPGO_KMS__SECRETS__URL="azurekeyvault://my-vault-name.vault.azure.net/keys/my-key-name"

SFTPGO_PLUGINS__0__TYPE=kms
SFTPGO_PLUGINS__0__KMS_OPTIONS__SCHEME=azurekeyvault
SFTPGO_PLUGINS__0__KMS_OPTIONS__ENCRYPTED_STATUS=AzureKeyVault
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-kms"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

You can optionally specify a key version: `azurekeyvault://my-vault.vault.azure.net/keys/my-key/version-id`.

Authentication uses the default Azure SDK credential chain, which automatically supports Azure Managed Identity, environment-based credentials (`AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`), and Azure CLI credentials.

### HashiCorp Vault

```shell
SFTPGO_KMS__SECRETS__URL="hashivault://my-key"

SFTPGO_PLUGINS__0__TYPE=kms
SFTPGO_PLUGINS__0__KMS_OPTIONS__SCHEME=hashivault
SFTPGO_PLUGINS__0__KMS_OPTIONS__ENCRYPTED_STATUS=VaultTransit
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-kms"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

Requires the following environment variables:

- `VAULT_SERVER_URL` — Vault server address
- `VAULT_SERVER_TOKEN` — authentication token

### Oracle Key Vault

```shell
SFTPGO_KMS__SECRETS__URL="ocikeyvault://ocid1.key.oc1..example"
SFTPGO_PLUGIN_KMS_OCI_ENDPOINT="https://example-crypto.kms.us-ashburn-1.oraclecloud.com"

SFTPGO_PLUGINS__0__TYPE=kms
SFTPGO_PLUGINS__0__KMS_OPTIONS__SCHEME=ocikeyvault
SFTPGO_PLUGINS__0__KMS_OPTIONS__ENCRYPTED_STATUS=OracleKeyVault
SFTPGO_PLUGINS__0__CMD="/usr/bin/sftpgo-plugin-kms"
SFTPGO_PLUGINS__0__AUTO_MTLS=1
```

By default, the plugin uses Instance Principal authentication (Oracle's managed identity). To use API key authentication instead, append `?auth_type_api_key=1` to the KMS secrets URL (e.g., `ocikeyvault://ocid1.key.oc1..example?auth_type_api_key=1`).

## Notes

- KMS configuration is global; all secrets in SFTPGo are encrypted with the same provider.
- You can migrate from the local provider to a cloud KMS provider, but you cannot switch between different cloud providers.
- An optional `master_key` can be set for dual encryption (local + cloud KMS). See [Key Management Services](../kms.md) for details.
