# Logs

SFTPGo logs a stream of JSON structs. Each struct has a `sender` field that identifies the log type.
SFTPGo logging options are set either by cli options or environment variables, not in the configuration file. More [details](cli.md#starting-the-server).

The logs can be divided into the following categories:

**app logs**, internal logs used to debug SFTPGo:

- `sender` string. This is generally the package name that emits the log
- `time` string. Date/time with millisecond precision
- `level` string
- `connection_id`, string, optional
- `message` string

**transfer logs**, SFTP/SCP transfer logs:

- `sender` string. `Upload` or `Download`
- `time` string. Date/time with millisecond precision
- `level` string
- `local_addr` string. IP/port of the local address the connection arrived on. For FTP protocol this is the address for the control connection. For example `127.0.0.1:1234`
- `remote_addr` string. IP and, optionally, port of the remote client. For example `127.0.0.1:1234` or `127.0.0.1`
- `elapsed_ms`, int64. Elapsed time, as milliseconds, for the upload/download
- `size_bytes`, int64. Size, as bytes, of the download/upload
- `username`, string
- `file_path` string
- `connection_id` string. Unique connection identifier
- `protocol` string. `SFTP`, `SCP`, `SSH`, `FTP`, `HTTP`, `HTTPShare`, `DAV`, `DataRetention`, `EventAction`
- `ftp_mode`, string. `active` or `passive`. Included only for `FTP` protocol
- `error`, string. Included if there is a transfer error

**command logs**, SFTP/SCP command logs:

- `sender` string. `Rename`, `Rmdir`, `Mkdir`, `Symlink`, `Remove`, `Chmod`, `Chown`, `Chtimes`, `Truncate`, `Copy`, `SSHCommand`
- `level` string-
- `local_addr` string. IP/port of the local address the connection arrived on. For example `127.0.0.1:1234`
- `remote_addr` string. IP and, optionally, port of the remote client. For example `127.0.0.1:1234` or `127.0.0.1`
- `username`, string
- `file_path` string
- `target_path` string
- `filemode` string. Valid for sender `Chmod` otherwise empty
- `uid` integer. Valid for sender `Chown` otherwise -1
- `gid` integer. Valid for sender `Chown` otherwise -1
- `access_time` datetime as YYYY-MM-DDTHH:MM:SS. Valid for sender `Chtimes` otherwise empty
- `modification_time` datetime as YYYY-MM-DDTHH:MM:SS. Valid for sender `Chtimes` otherwise empty
- `size` int64. Valid for sender `Truncate` otherwise -1
- `elapsed`, int64. Elapsed time, as milliseconds
- `ssh_command`, string. Valid for sender `SSHCommand` otherwise empty
- `connection_id` string. Unique connection identifier
- `protocol` string. `SFTP`, `SCP`, `SSH`, `FTP`, `HTTP`, `DAV`, `DataRetention`, `EventAction`

**http logs**, REST API logs:

- `sender` string. `httpd`
- `level` string
- `time` string. Date/time with millisecond precision
- `local_addr` string. IP/port of the local address the connection arrived on. For example `127.0.0.1:1234`
- `remote_addr` string. IP and, optionally, port of the remote client. For example `127.0.0.1:1234` or `127.0.0.1`
- `proto` string, for example `HTTP/1.1`
- `method` string. HTTP method (`GET`, `POST`, `PUT`, `DELETE` etc.)
- `request_id` string. Omitted in telemetry logs
- `user_agent` string
- `uri` string. Full uri
- `resp_status` integer. HTTP response status code
- `resp_size` integer. Size in bytes of the HTTP response
- `elapsed_ms` int64. Elapsed time, as milliseconds, to complete the request
- `request_id` string. Unique request identifier

**connection failed logs**, logs for failed attempts to initialize a connection. A connection can fail for an authentication error or other errors such as a client abort or a timeout

- `sender` string. `connection_failed`
- `level` string. `debug`
- `time` string. Date/time with millisecond precision
- `username`, string. Can be empty if the connection is closed before an authentication attempt
- `client_ip` string.
- `protocol` string. Possible values are `SSH`, `FTP`, `DAV`
- `login_type` string. Can be `publickey`, `password`, `keyboard-interactive`, `publickey+password`, `publickey+keyboard-interactive` or `no_auth_tried`
- `error` string. Optional error description

**login logs**, logs for successful logins

- `sender` string. `login`
- `level` string. `info`
- `time` string. Date/time with millisecond precision
- `username`, string.
- `ip` string.
- `protocol` string.
- `method` string.
- `connection_id` string. Optional.
- `client` string. Client software name if available.
- `encrypted`, boolean
- `info` string. Optional additional information.
