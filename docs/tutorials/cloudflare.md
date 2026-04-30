---
description: "Run SFTPGo behind Cloudflare with reliable browser uploads and accurate client IP addresses for the defender, rate limiter, and audit log."
---

# Running SFTPGo behind Cloudflare

When SFTPGo's WebClient or REST API is exposed through Cloudflare, two things benefit from a small amount of configuration:

1. **Large browser uploads** — Cloudflare proxies impose a per-request body size cap that varies by plan. A standard upload of a file larger than the cap fails. The fix is to enable **TUS** on the SFTPGo binding: a resumable, chunked upload protocol that splits the file into smaller pieces fitting under the cap and resumes automatically if a chunk fails. Consult Cloudflare's documentation for the exact cap on your plan.
2. **Real client IP addresses** — by default, every request reaching SFTPGo appears to come from a Cloudflare edge IP. The defender, rate limiter, audit log, and IP-based allow/deny lists all become useless because every connection looks identical. Telling SFTPGo to trust Cloudflare's `CF-Connecting-IP` header restores the real client IP.

You don't need to know the TUS protocol to use it: enable it on the binding and the WebClient picks it up automatically. REST API clients use it via libraries like [tus-js-client](https://github.com/tus/tus-js-client){:target="_blank"} or by calling the `/api/v2/user/files/chunked-upload` and `/api/v2/shares-chunked-uploads` endpoints — see the [REST API documentation](../rest-api.md) for the full request shape.

## What to set

Add the following environment variables to your SFTPGo deployment. They configure binding `0` — adjust the `BINDINGS__0__` index if you need to apply this configuration to a different HTTP binding.

```bash
SFTPGO_HTTPD__BINDINGS__0__PROXY_ALLOWED="173.245.48.0/20,103.21.244.0/22,103.22.200.0/22,103.31.4.0/22,141.101.64.0/18,108.162.192.0/18,190.93.240.0/20,188.114.96.0/20,197.234.240.0/22,198.41.128.0/17,162.158.0.0/15,104.16.0.0/13,104.24.0.0/14,172.64.0.0/13,131.0.72.0/22"
SFTPGO_HTTPD__BINDINGS__0__CLIENT_IP_PROXY_HEADER="CF-Connecting-IP"
SFTPGO_HTTPD__BINDINGS__0__UPLOAD_CHUNK_SIZE=10
```

Restart SFTPGo to apply. The rest of this page explains each line.

## TLS termination at Cloudflare

If your SFTPGo HTTPD binding runs on plain HTTP and Cloudflare terminates TLS at its edge (so traffic from Cloudflare to SFTPGo is HTTP), also set:

```bash
SFTPGO_HTTPD__BINDINGS__0__SECURITY__ENABLED=true
SFTPGO_HTTPD__BINDINGS__0__SECURITY__HTTPS_PROXY_HEADERS__0__KEY="X-Forwarded-Proto"
SFTPGO_HTTPD__BINDINGS__0__SECURITY__HTTPS_PROXY_HEADERS__0__VALUE="https"
```

This tells SFTPGo to treat requests carrying `X-Forwarded-Proto: https` as HTTPS even though the binding itself listens on plain HTTP, so that generated absolute URLs use `https://` and the `Secure` flag is set on cookies. Cloudflare adds this header on every proxied request, and `PROXY_ALLOWED` above already restricts who can set it. If your binding has its own TLS certificate (`enable_https = true`), skip this section.

## How it works

### Trusting Cloudflare's client IP header

`PROXY_ALLOWED` is the list of upstream addresses SFTPGo accepts a client-IP header from. When a request comes from one of these IPs, SFTPGo reads the header named in `CLIENT_IP_PROXY_HEADER` and uses its value as the real client IP. Headers from any other upstream are silently ignored, so an attacker connecting directly cannot spoof their IP.

The IP ranges above are Cloudflare's published IPv4 list. If your binding accepts IPv6 traffic, append the IPv6 ranges from [https://www.cloudflare.com/ips/](https://www.cloudflare.com/ips/){:target="_blank"} to `PROXY_ALLOWED` as well.

`CF-Connecting-IP` is the header Cloudflare adds to every proxied request, containing the real client IP. The standard `X-Forwarded-For` works too, but `CF-Connecting-IP` carries a single IP address (no chain to parse) and Cloudflare overwrites any client-supplied value.

:warning: Cloudflare updates its IP ranges occasionally. Check [https://www.cloudflare.com/ips/](https://www.cloudflare.com/ips/){:target="_blank"} or `https://api.cloudflare.com/client/v4/ips` periodically and update `PROXY_ALLOWED` if the list changes. An outdated list means SFTPGo silently ignores the header for affected sources.

### Enabling TUS uploads on the binding

`UPLOAD_CHUNK_SIZE` is the chunk size, in MB, that the WebClient uses to split files. A value greater than zero enables the TUS endpoints on this binding for both the WebClient and the REST API; `0` disables TUS and the WebClient falls back to single-request uploads.

When TUS is enabled, the WebClient splits each upload into chunks of the configured size and, within the server's idle timeout, resumes from the last completed chunk if a chunk fails (network drop, browser tab refresh, etc.).

REST API clients can keep using single-request uploads (`POST /api/v2/user/files` for users, `POST /api/v2/shares/{id}` for shares) — the TUS endpoints exist alongside, not instead. The TUS protocol does not prescribe a chunk size; programmatic clients pick whatever suits them and do not read `upload_chunk_size`.

:information_source: TUS applies only to the WebClient and REST API. WebDAV uploads use plain `PUT`, while SFTP, SCP, and FTPS have their own resume mechanisms.

## Verifying the configuration

After restart, log into the WebClient and upload a file larger than your Cloudflare body-size cap. In the browser's developer tools (Network tab), you should see a series of `PATCH` requests to `/web/client/chunked-upload/<id>` instead of a single multipart `POST`, each with a body at or below the configured chunk size.

To verify the client IP is being captured correctly, check the logs. The IP shown should be the real public IP of the client, not a Cloudflare edge IP from the ranges in `PROXY_ALLOWED`.

## Scope of these settings

The environment variables above target the SFTPGo HTTPD binding, which serves the **WebAdmin**, **WebClient**, and **REST API**. **SFTP** and **FTP/S** are not HTTP and do not pass through Cloudflare's HTTP proxy: they must reach SFTPGo directly, or be exposed through a TCP-level proxy.

## Multi-instance setups

In multi-instance deployments behind a load balancer, TUS upload state is held in memory on the instance that received the initial `POST`. Subsequent `PATCH` requests for the same upload must reach the same instance, otherwise they fail with `404`. Configure session affinity on your load balancer.

## Related pages

- [Configuration file reference](../config-file.md) — full description of `proxy_allowed`, `client_ip_proxy_header`, `upload_chunk_size`, and other binding options.
- [Environment variables](../env-vars.md) — how `SFTPGO_HTTPD__BINDINGS__0__*` env vars map to the configuration file.
- [Web interfaces](../web-interfaces.md) — WebAdmin and WebClient overview.
