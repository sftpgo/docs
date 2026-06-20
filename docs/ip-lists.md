---
description: "SFTPGo IP Lists explained: the Allow List for default-deny access control and the Trusted List for defender and rate-limiter exemptions. Enable, scope by protocol, and manage via WebAdmin or REST API."
---

# IP Lists

SFTPGo maintains permanent, manually curated lists of IP addresses and networks, managed from the WebAdmin under the **IP Manager** section or through the REST API (`/ip-lists`). They are independent of the [defender](defender.md), which builds its block list dynamically from failed logins.

The IP Manager exposes four distinct lists, each with its own purpose. Choosing the right one is the most common source of confusion, so start here:

| List | Purpose | Effect on a listed IP | Entry modes | How it activates |
| ---- | ------- | --------------------- | ----------- | ---------------- |
| **Allow List** | Access control (default-deny gate) | The IP is **permitted** to connect; every IP not on the list is rejected before authentication | Allow only | Enabled by `allowlist_status` |
| **Trusted List** | Exemption from protection layers | The IP **bypasses** the defender, rate limiters, and IP-filtering plugins (such as GeoIP) | Allow only | Active whenever it contains entries |
| **Defender List** | Static block / permit rules for the [defender](defender.md) | `Deny` always blocks; `Allow` always permits | Allow and Deny | Active whenever it contains entries |
| **Rate Limiters Safe List** | Exemption from [rate limiting](rate-limiting.md) only | The IP is never delayed or rejected by rate limiters | Allow only | Active whenever it contains entries |

Each entry can be a single IP address or a CIDR network (for example `203.0.113.0/24`), in IPv4 or IPv6, and can be scoped to specific protocols.

## Allow List

The Allow List implements **default-deny access control**: when enabled, only the IP addresses and networks present in the list can reach SFTPGo's services; every other connection is dropped at connection time, before authentication. It protects all services — SFTP/SCP, FTP, WebDAV, and the HTTP interfaces (WebAdmin, WebClient, REST API).

This is the right tool when you want to expose a service only to a known set of source IPs.

### Enabling the Allow List

The Allow List is consulted only when it is turned on through the `allowlist_status` setting. Set it in the configuration file:

```json
{
  "common": {
    "allowlist_status": 1
  }
}
```

or via environment variable:

```shell
SFTPGO_COMMON__ALLOWLIST_STATUS=1
```

`0` (the default) disables the feature; `1` enables it.

:warning: Populate the list with at least one allowed entry **before** enabling it, and make sure your own management IP is included. When the feature is enabled with an empty list, no address is listed, so every connection — including yours — is denied. Add the entries first, verify them, then set `allowlist_status` to `1`.

### Scoping by protocol

Each entry applies to all protocols by default, or to a subset you select (SSH, FTP, WebDAV, HTTP). For example, you can allow an automation host for SFTP only while keeping it out of the HTTP interfaces. An entry that allows an IP for SSH does not, on its own, allow that IP for WebDAV.

## Trusted List

The Trusted List is **not** an access-control mechanism — it never grants or denies access on its own. It is an *exemption* list: addresses on it bypass SFTPGo's protective layers. A trusted IP is:

- never banned by the [defender](defender.md), and its activity is not counted toward auto-blocking;
- never delayed or rejected by [rate limiters](rate-limiting.md);
- exempt from IP-filtering plugins, such as [GeoIP filtering](plugins/geoip-filter.md).

Use it for trusted networks or monitoring hosts that you never want throttled or banned. The list is active whenever it contains entries; it has no enable/disable switch and accepts allow-mode entries only.

:information_source: The Trusted List does not put an IP on the Allow List. If the Allow List is enabled, a trusted IP must also be present on the Allow List to be able to connect.

## Defender List and Rate Limiters Safe List

The **Defender List** holds static rules for the [defender](defender.md). Each entry has a mode: `Deny` always blocks the address, `Allow` always permits it (overriding a dynamic ban). When an address is covered by several entries, the most specific one — the longest network prefix — wins, so a narrow `Allow` correctly overrides a broad `Deny` such as `0.0.0.0/0`, and vice versa.

The **Rate Limiters Safe List** exempts addresses from [rate limiting](rate-limiting.md) only, without affecting the defender or plugins. It is a narrower alternative to the Trusted List when you want to relax rate limits without disabling other protections.

## Managing entries

Manage all four lists from the WebAdmin **IP Manager → IP Lists** page, selecting the list type from the dropdown, or through the REST API at `/ip-lists` (see the [REST API reference](https://sftpgo.com/rest-api){:target="_blank"}). Each entry stores:

- **IP/Network** — a single address or a CIDR network, IPv4 or IPv6;
- **Mode** — `Allow` or `Deny` (`Deny` is available on the Defender List only);
- **Protocols** — the protocols the entry applies to, or all of them;
- **Description** — an optional note.

## Example: default-deny SFTP from a curated list

To allow SFTP access only from a fixed set of source IPs:

1. In the WebAdmin, open **IP Manager → IP Lists**, select **Allow list**, and add each approved address or network, scoped to `SSH` if you only want to restrict SFTP/SCP.
2. Add an entry for the IP you administer SFTPGo from, scoped to `HTTP` (or to all protocols). The Allow List also gates the HTTP interfaces, so without this entry, enabling the feature locks you out of the WebAdmin and REST API. If you administer over SFTP from a different host, allow that host too.
3. Confirm all entries appear in the list.
4. Enable the feature by setting `allowlist_status` to `1` (config file or `SFTPGO_COMMON__ALLOWLIST_STATUS=1`) and reload SFTPGo.

From this point, connections from any source outside the list are dropped before authentication, while the listed sources connect on the protocols you granted them. To lift the restriction, set `allowlist_status` back to `0`.

:warning: Because the Allow List gates the HTTP interfaces too, always keep an entry that permits your administration IP on `HTTP`. Otherwise you will not be able to reach the WebAdmin or REST API to fix the list, and will have to disable `allowlist_status` from the configuration and restart SFTPGo to recover.
