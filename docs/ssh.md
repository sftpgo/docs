# SSH

SFTPGo is mainly an SFTP server only a minimal set of SSH commands are supported. Shell login and forwarding are not currently supported.

## SFTP

The SFTP implementation supports the SFTP sever protocol [version 3](https://datatracker.ietf.org/doc/html/draft-ietf-secsh-filexfer-02){:target="_blank"}, the same as OpenSSH.

## SSH commands

Some SSH commands are implemented directly inside SFTPGo, while for others we use system commands that need to be installed and in your system's `PATH`.

For system commands we have no direct control on file creation/deletion and so there are some limitations:

- we cannot allow them if the target directory contains virtual folders or file extensions filters
- system commands work only on local filesystem
- we cannot avoid to leak real filesystem paths
- quota check is suboptimal
- maximum size restriction on single file is not respected
- data at-rest encryption is not supported

 If quota is enabled and SFTPGo receives a system command, the used size and number of files are checked at the command start and not while new files are created/deleted. While the command is running the number of files is not checked, the remaining size is calculated as the difference between the max allowed quota and the used one, and it is checked against the bytes transferred via SSH. The command is aborted if it uploads more bytes than the remaining allowed size calculated at the command start. Anyway, we only see the bytes that the remote command sends to the local one via SSH. These bytes contain both protocol commands and files, and so the size of the files is different from the size transferred via SSH: for example, a command can send compressed files, or a protocol command (few bytes) could delete a big file. To mitigate these issues, quotas are recalculated at the command end with a full scan of the directory specified for the system command. This could be heavy for big directories. If you need system commands and quotas you could consider disabling quota restrictions and periodically update quota usage yourself using the REST API.

 :warning: SFTPGo attempts to sanitize system command arguments and restrict their use to the intended use cases. But these are external programs that could be used in many ways, and this could lead to unexpected behavior or even security issues. Please consider carefully before enabling the supported system commands.

 For these reasons we should limit system commands usage as much as possible, we currently support the following system commands:

- `rsync`. The `rsync` command needs to be installed and in your system's `PATH`.

At least the following permissions are required to be able to run system commands:

- `list`
- `download`
- `upload`
- `create_dirs`
- `overwrite`
- `delete`

For `rsync`  we cannot avoid that it creates symlinks so if the `create_symlinks` permission is granted we add the option `--safe-links`, if it is not already set, to the received `rsync` command. This should prevent to create symlinks that point outside the home directory.
If the user cannot create symlinks we add the option `--munge-links`, if it is not already set, to the received `rsync` command. This should make symlinks unusable (but manually recoverable).

**Note**: you might consider to use SFTPGo as SFTP backend for [rclone](https://rclone.org/sftp/){:target="_blank"}, or similar software that allows to synchronize files via SFTP, instead of `rsync`, this way there are no limitations and `rclone` does not need to be installed on the server side since it uses the SFTP protocol. Suppprt for `rsync` may be removed in the future.

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
