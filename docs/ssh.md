---
description: "Open-source SFTPGo SFTP server with post-quantum cryptography: ML-KEM hybrid key exchange, ML-DSA host and user keys, SSH commands."
---

# SSH

SFTPGo is mainly an SFTP server only a minimal set of SSH commands are supported. Shell login and forwarding are not currently supported.

## SFTP

The SFTP implementation supports the SFTP sever protocol [version 3](https://datatracker.ietf.org/doc/html/draft-ietf-secsh-filexfer-02){:target="_blank"}, the same as OpenSSH.

## Public key authentication

A user authenticates with any of the public keys stored on the account, in the `authorized_keys` format. The options of that format, `from`, `command`, `no-pty` and the others, are accepted and have no effect: a key with a `from` option authenticates from any address. To limit where a user may connect from, use the allowed and denied IP addresses of the user; for a per-key restriction, issue an SSH certificate with the `source-address` option instead of storing the key.

## Post-quantum cryptography

SFTPGo uses post-quantum algorithms in both the key exchange and the signatures of the SSH protocol:

- **Key exchange**: `mlkem768x25519-sha256`, ML-KEM-768 combined with X25519, enabled by default and first in the default list.
- **Signatures**: host and user keys can use ML-DSA (`ssh-mldsa-44`, `ssh-mldsa-65`, `ssh-mldsa-87`), enabled by default.

The client picks, in its own order of preference, among the algorithms both sides support: clients without post-quantum support use the classical ones.

An ML-DSA host key is read in PKCS#8 PEM format holding the seed only. With OpenSSL 3.5 or later:

```shell
openssl genpkey -algorithm ML-DSA-65 -out mldsa.pem
openssl pkey -in mldsa.pem -provparam ml-dsa.output_formats=seed-only -out id_mldsa65
```

Then add `id_mldsa65` to [`host_keys`](config-file.md#sshsftp-server), together with the keys already in use: `id_rsa`, `id_ecdsa` and `id_ed25519` for the generated ones.

## SSH commands

SFTPGo supports the following built-in SSH commands:

- `scp`, SFTPGo implements the SCP protocol so we can support it for cloud filesystems too and we can avoid the other system commands limitations. SCP between two remote hosts is supported using the `-3` scp option. Wildcard expansion is not supported.
- `md5sum`, `sha1sum`, `sha256sum`, `sha384sum`, `sha512sum`. Useful to check message digests for uploaded files.
- `cd`, `pwd`. Some SFTP clients do not support the SFTP SSH_FXP_REALPATH packet type, so they use `cd` and `pwd` SSH commands to get the initial directory. Currently `cd` does nothing and `pwd` always returns the `/` path.
- `sftpgo-copy`. This is a built-in copy implementation. It allows server side copy for files and directories. The first argument is the source file/directory and the second one is the destination file/directory, for example `sftpgo-copy <src> <dst>`.
- `sftpgo-remove`. This is a built-in remove implementation. It allows to remove single files and to recursively remove directories. The first argument is the file/directory to remove, for example `sftpgo-remove <dst>`. Removing directories spanning virtual folders is not supported.

:warning: The hash commands read the whole file to compute the digest: for remote backends this means downloading it, for the encrypted backend this means decrypting it. The read requires the `download` permission on the file's parent directory.

The following SSH commands are enabled by default:

- `md5sum`
- `sha1sum`
- `sha256sum`
- `cd`
- `pwd`
- `scp`
