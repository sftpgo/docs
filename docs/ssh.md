---
description: "Open-source SFTPGo SSH/SFTP server: host keys, ciphers, KEX algorithms, public key authentication, and post-quantum hybrid key exchange (mlkem768x25519)."
---

# SSH

SFTPGo is mainly an SFTP server only a minimal set of SSH commands are supported. Shell login and forwarding are not currently supported.

## SFTP

The SFTP implementation supports the SFTP sever protocol [version 3](https://datatracker.ietf.org/doc/html/draft-ietf-secsh-filexfer-02){:target="_blank"}, the same as OpenSSH.

## SSH commands

SFTPGo supports the following built-in SSH commands:

- `scp`, SFTPGo implements the SCP protocol so we can support it for cloud filesystems too and we can avoid the other system commands limitations. SCP between two remote hosts is supported using the `-3` scp option. Wildcard expansion is not supported.
- `md5sum`, `sha1sum`, `sha256sum`, `sha384sum`, `sha512sum`. Useful to check message digests for uploaded files.
- `cd`, `pwd`. Some SFTP clients do not support the SFTP SSH_FXP_REALPATH packet type, so they use `cd` and `pwd` SSH commands to get the initial directory. Currently `cd` does nothing and `pwd` always returns the `/` path.
- `sftpgo-copy`. This is a built-in copy implementation. It allows server side copy for files and directories. The first argument is the source file/directory and the second one is the destination file/directory, for example `sftpgo-copy <src> <dst>`.
- `sftpgo-remove`. This is a built-in remove implementation. It allows to remove single files and to recursively remove directories. The first argument is the file/directory to remove, for example `sftpgo-remove <dst>`. Removing directories spanning virtual folders is not supported.

:warning: The hash commands read the whole file to compute the digest: for remote backends this means downloading it, for the encrypted backend this means decrypting it. The read goes through a regular download transfer and requires the `download` permission on the file's parent directory.

The following SSH commands are enabled by default:

- `md5sum`
- `sha1sum`
- `sha256sum`
- `cd`
- `pwd`
- `scp`
