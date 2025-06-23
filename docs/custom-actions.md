# Custom Actions

SFTPGo can notify filesystem and provider events using custom actions. A custom action can be an external program or an HTTP URL.

## Filesystem events

The `actions` struct inside the `common` configuration section allows to configure the actions for file operations and SSH commands.
The `hook` can be defined as the absolute path of your program or an HTTP URL.

The following `actions` are supported:

- `download`
- `first-download`
- `pre-download`
- `upload`
- `first-upload`
- `pre-upload`
- `delete`
- `pre-delete`
- `rename`
- `mkdir`
- `rmdir`
- `ssh_cmd`
- `copy`

The `upload` condition includes both uploads to new files and overwrite of existing ones. If an upload is aborted for quota limits SFTPGo tries to remove the partial file, so if the notification reports a zero size file and a quota exceeded error the file has been deleted. The `ssh_cmd` condition will be triggered after a command is successfully executed via SSH. `scp` will trigger the `download` and `upload` conditions and not `ssh_cmd`. The `first-download` and `first-upload` action are executed only if no error occour and they don't exclude the `download` and `upload` notifications, so you will get both the `first-upload` and `upload` notification after the first successful upload and the same for the first successful download.
For cloud backends directories are virtual, they are created implicitly when you upload a file and are implicitly removed when the last file within a directory is removed. The `mkdir` and `rmdir` notifications are sent only when a directory is explicitly created or removed.

The notification will indicate if an error is detected and so, for example, a partial file is uploaded.

The `pre-delete`, `pre-download` and `pre-upload` actions, will be called before deleting, downloading and uploading files. If the external command completes with a zero exit status or the HTTP notification response code is `200`, SFTPGo will allow the operation, otherwise the client will get a permission denied error.

If the `hook` defines a path to an external program, then this program can read the following environment variables:

- `SFTPGO_ACTION`, supported action
- `SFTPGO_ACTION_USERNAME`
- `SFTPGO_ACTION_PATH`, is the full filesystem path, can be empty for some ssh commands
- `SFTPGO_ACTION_TARGET`, full filesystem path, non-empty for `rename` `SFTPGO_ACTION` and for some SSH commands
- `SFTPGO_ACTION_VIRTUAL_PATH`, virtual path, seen by SFTPGo users
- `SFTPGO_ACTION_VIRTUAL_TARGET`, virtual target path, seen by SFTPGo users
- `SFTPGO_ACTION_SSH_CMD`, non-empty for `ssh_cmd` `SFTPGO_ACTION`
- `SFTPGO_ACTION_FILE_SIZE`, non-zero for `pre-upload`, `upload`, `download`, `delete`, and `copy` actions if the file size is greater than `0`
- `SFTPGO_ACTION_ELAPSED`, elapsed time as milliseconds
- `SFTPGO_ACTION_FS_PROVIDER`, `0` for local filesystem, `1` for S3 backend, `2` for Google Cloud Storage (GCS) backend, `3` for Azure Blob Storage backend, `4` for local encrypted backend, `5` for SFTP backend
- `SFTPGO_ACTION_BUCKET`, non-empty for S3, GCS and Azure backends
- `SFTPGO_ACTION_ENDPOINT`, non-empty for S3, SFTP and Azure backend if configured
- `SFTPGO_ACTION_STATUS`, integer. Status for `upload`, `download` and `ssh_cmd` actions. 1 means no error, 2 means a generic error occurred, 3 means quota exceeded error
- `SFTPGO_ACTION_PROTOCOL`, string. Possible values are `SSH`, `SFTP`, `SCP`, `FTP`, `DAV`, `HTTP`, `HTTPShare`, `OIDC`, `DataRetention`, `EventAction`
- `SFTPGO_ACTION_IP`, the action was executed from this IP address
- `SFTPGO_ACTION_SESSION_ID`, string. Unique protocol session identifier. For stateless protocols such as HTTP the session id will change for each request
- `SFTPGO_ACTION_OPEN_FLAGS`, integer. File open flags, can be non-zero for `pre-upload` action. If `SFTPGO_ACTION_FILE_SIZE` is greater than zero and `SFTPGO_ACTION_OPEN_FLAGS&512 == 0` the target file will not be truncated
- `SFTPGO_ACTION_ROLE`, string. Role of the user who executed the action
- `SFTPGO_ACTION_TIMESTAMP`, int64. Event timestamp as nanoseconds since epoch
- `SFTPGO_ACTION_METADATA`, string. Object metadata serialized as JSON. Omitted if there is no metadata

Global environment variables are cleared, for security reasons, when the script is called. You can set additional environment variables in the "command" configuration section.
The program must finish within 30 seconds.

If the `hook` defines an HTTP URL then this URL will be invoked as HTTP POST. The request body will contain a JSON serialized struct with the following fields:

- `action`, string
- `username`, string
- `path`, string
- `target_path`, string, included for `rename` action and `sftpgo-copy` SSH command
- `virtual_path`, string, virtual path, seen by SFTPGo users
- `virtual_target_path`, string, virtual target path, seen by SFTPGo users
- `ssh_cmd`, string, included for `ssh_cmd` action
- `file_size`, int64, included for `pre-upload`, `upload`, `download`, `delete` and `copy` actions if the file size is greater than `0`
- `elapsed`, int64, elapsed size as milliseconds
- `fs_provider`, integer, `0` for local filesystem, `1` for S3 backend, `2` for Google Cloud Storage (GCS) backend, `3` for Azure Blob Storage backend, `4` for local encrypted backend, `5` for SFTP backend, `6` for HTTPFs backend
- `bucket`, string, included for S3, GCS and Azure backends
- `endpoint`, string, included for S3, SFTP and Azure backend if configured
- `status`, integer. Status for `upload`, `download` and `ssh_cmd` actions. 1 means no error, 2 means a generic error occurred, 3 means quota exceeded error
- `protocol`, string. Possible values are `SSH`, `SFTP`, `SCP`, `FTP`, `DAV`, `HTTP`, `HTTPShare`, `OIDC`, `DataRetention`, `EventAction`
- `ip`, string. The action was executed from this IP address
- `session_id`, string. Unique protocol session identifier. For stateless protocols such as HTTP the session id will change for each request
- `open_flags`, integer. File open flags, can be non-zero for `pre-upload` action. If `file_size` is greater than zero and `file_size&512 == 0` the target file will not be truncated
- `role`, string. Included if the user who executed the action has a role
- `timestamp`, int64. Event timestamp as nanoseconds since epoch
- `metadata`, struct. Object metadata. Both the keys and the values are string. Omitted if there is no metadata

The HTTP hook will use the global configuration for HTTP clients and will respect the retry, TLS and headers configurations. See the HTTP Clients (`http`) section of the [config reference](config-file.md#http-clients).

The `pre-*` actions are always executed synchronously while the other ones are asynchronous. You can specify the actions to run synchronously via the `execute_sync` configuration key. Executing an action synchronously means that SFTPGo will not return a result code to the client (which is waiting for it) until your hook have completed its execution. If your hook takes a long time to complete this could cause a timeout on the client side, which wouldn't receive the server response in a timely manner and eventually drop the connection.
If you add the `upload` action to the `execute_sync` configuration key, SFTPGo will try to delete the uploaded file and return an error to the client if the hook fails. A hook is considered failed if the external command completes with a non-zero exit status or the HTTP notification response code is other than `200` (or the HTTP endpoint cannot be reached or times out).
After a hook failure, the uploaded size is removed from the quota if SFTPGo is able to remove the file.

## Provider events

The `actions` struct inside the `data_provider` configuration section allows you to configure actions on data provider objects add, update, delete.

The supported object types are:

- `user`
- `folder`
- `group`
- `admin`
- `api_key`
- `share`
- `event_action`
- `event_rule`

Actions will not be fired for internal updates, such as the last login or the user quota fields, or after external authentication.

If the `hook` defines a path to an external program, then this program can read the following environment variables:

- `SFTPGO_PROVIDER_ACTION`, supported values are `add`, `update`, `delete`
- `SFTPGO_PROVIDER_OBJECT_TYPE`, affected object type
- `SFTPGO_PROVIDER_OBJECT_NAME`, unique identifier for the affected object, for example username or key id
- `SFTPGO_PROVIDER_USERNAME`, the admin username that executed the action. There are two special usernames: `__self__` identifies a user/admin that updates itself and `__system__` identifies an action that does not have an explicit executor associated with it, for example users/admins can be added/updated by loading them from initial data
- `SFTPGO_PROVIDER_IP`, the action was executed from this IP address
- `SFTPGO_PROVIDER_ROLE`, the action was executed by an admin with this role
- `SFTPGO_PROVIDER_TIMESTAMP`, event timestamp as nanoseconds since epoch
- `SFTPGO_PROVIDER_OBJECT`, object serialized as JSON with sensitive fields removed

Global environment variables are cleared, for security reasons, when the script is called. You can set additional environment variables in the "command" configuration section.
The program must finish within 15 seconds.

If the `hook` defines an HTTP URL then this URL will be invoked as HTTP POST. The action, username, ip, object_type and object_name and timestamp and role are added to the query string, for example `<hook>?action=update&username=admin&ip=127.0.0.1&object_type=user&object_name=user1&timestamp=1633860803249`, and the full object is sent serialized as JSON inside the POST body with sensitive fields removed. The role is added only if not empty.

The HTTP hook will use the global configuration for HTTP clients and will respect the retry, TLS and headers configurations. See the HTTP Clients (`http`) section of the [config reference](config-file.md#http-clients).

The structure for SFTPGo objects can be found within the [OpenAPI schema](https://sftpgo.com/rest-api){:target="_blank"}.

## Pub/Sub services

You can forward SFTPGo events to several publish/subscribe systems using the [sftpgo-plugin-pubsub](https://github.com/sftpgo/sftpgo-plugin-pubsub){:target="_blank"}. The notifiers SFTPGo plugins are not suitable for interactive actions such as `pre-*` events. Their scope is to simply forward events to external services. A custom hook is a better choice if you need to react to `pre-*` events.

## Database services

You can store SFTPGo events in database systems using the [sftpgo-plugin-eventstore](https://github.com/sftpgo/sftpgo-plugin-eventstore){:target="_blank"} and you can search the stored events using the [sftpgo-plugin-eventsearch](https://github.com/sftpgo/sftpgo-plugin-eventsearch){:target="_blank"}.
