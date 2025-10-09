# SSH

SFTPGo is mainly an SFTP server only a minimal set of SSH commands are supported. Shell login and forwarding are not currently supported.

## SFTP

The SFTP implementation supports the SFTP sever protocol [version 3](https://datatracker.ietf.org/doc/html/draft-ietf-secsh-filexfer-02){:target="_blank"}, the same as OpenSSH.

## SSH commands

SFTPGo supports the following built-in SSH commands:

- `scp`, SFTPGo implements the SCP protocol so we can support it for cloud filesystems too and we can avoid the other system commands limitations. SCP between two remote hosts is supported using the `-3` scp option. Wildcard expansion is not supported.
- `md5sum`, `sha1sum`, `sha256sum`, `sha384sum`, `sha512sum`. Useful to check message digests for uploaded files.
- `cd`, `pwd`. Some SFTP clients do not support the SFTP SSH_FXP_REALPATH packet type, so they use `cd` and `pwd` SSH commands to get the initial directory. Currently `cd` does nothing and `pwd` always returns the `/` path. These commands will work with any storage backend but keep in mind that to calculate the hash we need to read the whole file, for remote backends this means downloading the file, for the encrypted backend this means decrypting the file.
- `sftpgo-copy`. This is a built-in copy implementation. It allows server side copy for files and directories. The first argument is the source file/directory and the second one is the destination file/directory, for example `sftpgo-copy <src> <dst>`.
- `sftpgo-remove`. This is a built-in remove implementation. It allows to remove single files and to recursively remove directories. The first argument is the file/directory to remove, for example `sftpgo-remove <dst>`. Removing directories spanning virtual folders is not supported.

The following SSH commands are enabled by default:

- `md5sum`
- `sha1sum`
- `sha256sum`
- `cd`
- `pwd`
- `scp`
