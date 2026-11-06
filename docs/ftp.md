---
description: "Open-source SFTPGo FTP/FTPS server configuration: explicit and implicit TLS, passive mode, and client certificates."
---

# FTP

The FTP server implementation supports [RFC 959](https://datatracker.ietf.org/doc/html/rfc959){:target="_blank"}.

Both password and client certificate authentication are supported.

The following extensions are implemented:

- [AUTH](https://tools.ietf.org/html/rfc2228#page-6){:target="_blank"} - Control session protection
- [AUTH TLS](https://tools.ietf.org/html/rfc4217#section-4.1){:target="_blank"} - TLS session
- [PROT](https://tools.ietf.org/html/rfc2228#page-8){:target="_blank"} - Transfer protection
- [EPRT/EPSV](https://tools.ietf.org/html/rfc2428){:target="_blank"} - IPv6 support
- [MDTM](https://tools.ietf.org/html/rfc3659#page-8){:target="_blank"} - File Modification Time
- [SIZE](https://tools.ietf.org/html/rfc3659#page-11){:target="_blank"} - Size of a file
- [REST](https://tools.ietf.org/html/rfc3659#page-13){:target="_blank"} - Restart of interrupted transfer
- [MLST](https://tools.ietf.org/html/rfc3659#page-23){:target="_blank"} - Simple file listing for machine processing
- [MLSD](https://tools.ietf.org/html/rfc3659#page-23){:target="_blank"} - Directory listing for machine processing
- [HASH](https://tools.ietf.org/html/draft-bryan-ftpext-hash-02){:target="_blank"} - Hashing of files
- [AVLB](https://tools.ietf.org/html/draft-peterson-streamlined-ftp-command-extensions-10#section-4){:target="_blank"} - Available space

## Passive port multiplexing

By default each passive transfer listens on its own port of the range, so the range must include a port for every passive transfer running at the same time. With `passive_port_multiplexing` set to `true`, a few ports are enough: the ports of `passive_port_range` are shared among clients, and each data connection is routed to its client by source IP address. Enable it when you cannot expose a range that large, otherwise keep the dedicated ports.

**How many ports to expose.** A port is busy only from a PASV/EPSV until the data connection that follows it is established, not for the whole transfer. While it is busy, other clients with the same IP address cannot use it, clients with different IP addresses can. The range must therefore include a port for each PASV/EPSV sent at the same time by clients that share an IP address, such as parallel transfers or clients behind the same NAT. When every port is busy for an IP address, the PASV/EPSV command fails.

Example with five exposed ports:

```shell
SFTPGO_FTPD__BINDINGS__0__PORT=2121
SFTPGO_FTPD__PASSIVE_PORT_RANGE__START=50000
SFTPGO_FTPD__PASSIVE_PORT_RANGE__END=50004
SFTPGO_FTPD__PASSIVE_PORT_MULTIPLEXING=true
```

- It requires a `passive_port_range`, a single FTP binding and `passive_connections_security` set to `0`. The FTP service does not start otherwise.
- The ports of the range keep listening while the FTP service runs: set the range to the ports exposed for passive connections only.
- The [PROXY protocol](config-file.md#proxy_protocol-modes) is supported for the data connections too.

**Security.** Clients that share an IP address, for example behind the same NAT, can receive each other's data connections: a client that connects first to the port reserved for another client with the same IP address gets its data connection, so it can read the files the other client downloads, or send the content of the files it uploads. This is a known FTP issue that also affects dedicated passive ports, but a small range makes the port easy to guess. Encrypting the data connections does not prevent it, so enable multiplexing only if your clients do not share IP addresses.
