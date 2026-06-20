---
description: "Use a remote SFTP server as storage in open-source SFTPGo, with optional buffering for improved performance."
---

# SFTP as storage backend

An SFTP account on another server can be used as storage for an SFTPGo account, so the remote SFTP server can be accessed in a similar way to the local file system.

Here are the supported configuration parameters:

- `Endpoint`, ssh endpoint as `host:port`
- `Username`
- `Password`
- `PrivateKey`
- `Fingerprints`
- `Prefix`
- `BufferSize`

The mandatory parameters are the endpoint, the username and a password or a private key. If you define both a password and a private key the key is tried first. The provided private key should be PEM encoded, something like this:

```shell
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACA8LWc4SahqKkAr4L3rS19w1Vt8/IAf4th2FZmf+PJ/vwAAAJBvnZIJb52S
CQAAAAtzc2gtZWQyNTUxOQAAACA8LWc4SahqKkAr4L3rS19w1Vt8/IAf4th2FZmf+PJ/vw
AAAEBE6F5Az4wzNfNYLRdG8blDwvPBYFXE8BYDi4gzIhnd9zwtZzhJqGoqQCvgvetLX3DV
W3z8gB/i2HYVmZ/48n+/AAAACW5pY29sYUBwMQECAwQ=
-----END OPENSSH PRIVATE KEY-----
```

The password and the private key are stored as ciphertext according to your [KMS configuration](kms.md).

SHA256 fingerprints for remote server host keys are optional but highly recommended: if you provide one or more fingerprints the server host key will be verified against them and the connection will be denied if none of the fingerprints provided match that for the server host key.

Specifying a prefix you can scope the account to a given path within the remote SFTP server — see [Path confinement](#path-confinement) below for what this boundary guarantees. If you set a prefix make sure it is neither a symbolic link itself nor located inside a symlinked directory.

Buffering can be enabled by setting a buffer size (in MB) greater than 0. By enabling buffering, the reads and writes, from/to the remote SFTP server, are split in multiple concurrent requests and this allows data to be transferred at a faster rate, over high latency networks, by overlapping round-trip times. With buffering enabled, resuming uploads and truncate are not supported and a file cannot be opened for both reading and writing at the same time. 0 means disabled.

Some SFTP servers (eg. AWS Transfer) do not support opening files read/write at the same time, you can enable buffering to work with them.

## Path confinement

When you set a prefix other than `/`, SFTPGo scopes the account to that path and its subdirectories on the remote server. In that case — and only then — SFTPGo resolves symbolic links in every path component as a best-effort check, so a link that persistently points outside the prefix is rejected rather than followed. The per-directory permission interaction described in [Symbolic links and permissions](config-file.md#symbolic-links-and-permissions) applies here too.

Keep in mind what this boundary does and does not guarantee:

- The prefix is layered on a remote account that SFTPGo reaches with the configured credentials, which can access the whole account. The prefix scopes the SFTPGo user to a subtree; it is not a separate account.
- Enforcement depends on SFTPGo resolving paths on the remote, so it assumes the remote server reports file types and link targets faithfully. On a remote server you do not control, treat the prefix as a scoping convenience, not a hard isolation boundary. For strict isolation between tenants, prefer a separate remote account per tenant over different prefixes on a shared account.
- The path is resolved and then operated on in separate steps, and the SFTP protocol has no atomic rooted operation, so a residual time-of-check/time-of-use window remains that cannot be fully closed on a remote backend: a client holding `create_symlinks` could, by winning a race, have a link followed past the prefix. Any such access stays within the remote account the configured credentials already reach. To remove this window on a remote you control, start from a tree with no symbolic links and keep symbolic-link creation disabled (the default `symlink_mode`); for isolation between tenants, prefer a separate account over a prefix (see above).

With the default prefix `/`, the whole remote account is the root: SFTPGo performs no per-component symlink resolution and the remote server's own boundary is what applies.
