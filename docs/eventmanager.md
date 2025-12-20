# Event Manager

The EventManager is a powerful feature that enables administrators to define automated responses based on specific events occurring within the system (e.g., file uploads, user creation, scheduled tasks, and more).

Rules are the conditions that determine when an action should be executed. Each rule specifies:

- Which event(s) it applies to (e.g., file uploaded, file downloaded, etc.)
- Filters to narrow down the triggering events (e.g., only for a particular user or directory)
- Which action(s) to execute when the rule matches

Think of a rule as a "when this happens, and these conditions are met, then do that" type of logic.

Actions are the tasks performed when a rule is triggered. These actions can be dynamically customized using placeholders—variables that represent contextual data related to the event (such as file name, username, or file size). To further tailor these values, SFTPGo provides helper functions that format or transform placeholders directly within your action templates.

## Rules

Event rules are based on the premise that an event occours. To each rule you can associate one or more actions.
The following trigger events are supported:

- `Filesystem events`, for example `upload`, `download` etc.
- `Provider events`, for example `add`, `update`, `delete` user or other resources.
- `Schedules`. The scheduler uses UTC time.
- `IP Blocked`, this event can be generated if you enable the [defender](./defender.md).
- `Certificate`, this event is generated when a certificate is renewed using the built-in ACME protocol. Both successful and failed renewals are notified.
- `On demand`, this trigger is generated manually using the WebAdmin or the REST API.
- `Identity Provider login`, this trigger is generated when a user/admin logs in using an external Identity Provider.

You can further restrict a rule by specifying additional conditions that must be met before the rule’s actions are taken. For example you can react to uploads only if they are performed by a particular user or using a specified protocol.

Actions such as user quota reset, transfer quota reset, data retention check, folder quota reset and filesystem events are executed for all matching users if the trigger is a schedule or for the affected user if the trigger is a provider event or a filesystem action.

Actions are executed in a sequential order except for sync actions that are executed before the others. For each action associated to a rule you can define the following settings:

- `Stop on failure`, the next action will not be executed if the current one fails.
- `Failure action`, this action will be executed only if at least another one fails. :warning: Please note that a failure action isn't executed if the event fails, for example if a download fails the main action is executed. The failure action is executed only if one of the non-failure actions associated to a rule fails.
- `Execute sync`, for upload events, you can execute the action(s) synchronously. Executing an action synchronously means that SFTPGo will not return a result code to the client (which is waiting for it) until your action have completed its execution. If your acion takes a long time to complete this could cause a timeout on the client side, which wouldn't receive the server response in a timely manner and eventually drop the connection. For pre-* events at least a sync action is required. If pre-delete,pre-upload, pre-download sync action(s) completes successfully, SFTPGo will allow the operation, otherwise the client will get a permission denied error.

If you are running multiple SFTPGo instances connected to the same data provider, you can choose whether to allow simultaneous execution for scheduled actions.

Some actions are not supported for some triggers, rules containing incompatible actions are skipped at runtime:

- `Filesystem events`, folder quota reset cannot be executed, we don't have a direct way to get the affected folder.
- `Provider events`, user quota reset, transfer quota reset, data retention check and filesystem actions can be executed only if  a user is updated. They will be executed for the affected user. Folder quota reset can be executed only for folders. Filesystem actions are not executed for `delete` user events because the actions is executed after the user deletion.
- `IP Blocked`, user quota reset, folder quota reset, transfer quota reset, data retention check and filesystem actions cannot be executed, we only have an IP.
- `Certificate`, user quota reset, folder quota reset, transfer quota reset, data retention check and filesystem actions cannot be executed.
- `Email with attachments` are supported for filesystem events and provider events if a user is added/updated. We need a user to get the files to attach.
- `HTTP multipart requests with files as attachments` are supported for filesystem events and provider events if a user is added/updated. We need a user to get the files to attach.

## Actions

Supported actions:

- `HTTP notification`. You can notify an HTTP/S endpoing via GET, POST, PUT, DELETE methods. You can define custom headers, query parameters and a body for POST and PUT request. Placeholders are supported for username, body, header and query parameter values.
- `Command execution`. You can launch custom commands passing parameters via environment variables. Placeholders are supported for environment variable values. :warning: Allowing any system command could pose a security risk, they are disabled by default.
- `Email notification`. Placeholders are supported in subject and body. The email will be sent as plain text. For this action to work you have to configure an SMTP server in the SFTPGo configuration file.
- `Backup`. A backup will be saved in the configured backup directory. The backup will contain the week day, the hour and the minute in the file name.
- `Rotate log file`. If file logging is enabled, the log file will be rotated regardless of its size.
- `User quota reset`. The quota used by users will be updated based on current usage.
- `Folder quota reset`. The quota used by virtual folders will be updated based on current usage.
- `Transfer quota reset`. The transfer quota values will be reset to `0`.
- `Data retention check`. You can define per-folder retention policies.
- `Password expiration check`. You can send an email notification to users whose password is about to expire.
- `User expiration check`. You can receive notifications with expired users.
- `User inactivity check`. Allow to disable or delete inactive users.
- `Identity Provider account check`. You can create/update accounts for users/admins logging in using an Identity Provider.
- `Filesystem`. For these actions, the required permissions are automatically granted. This is the same as executing the actions from an SFTP client and the same restrictions applies. Supported actions:
  - `Rename`. You can rename one or more files or directories.
  - `Delete`. You can delete one or more files and directories.
  - `Create directories`. You can create one or more directories including sub-directories.
  - `Path exists`. Check if the specified path exists.
  - `Copy`. You can copy one or more files or directories.
  - `Compress`. You can compress (currently as zip) ore or more files and directories.
  - `Extract`. Allows extraction of ZIP archives. To mitigate common ZIP-based attacks, several limits are enforced, which can be adjusted via [environment variables](env-vars.md):
    - Maximum allowed compression ratio: 60.
    - Maximum number of files: 1000.
    - Maximum uncompressed size: 1 GB.
  - `PGP` encryption and decryption, allowing you to secure your files using either password-based encryption or PGP key pairs. This includes the ability to sign and verify digital signatures, ensuring both the authenticity and integrity of your data throughout the process.
  - `Metadata Check` verifies whether a specified metadata key exists and matches a configured value, or whether it is absent. To check for non-existence, leave the value field empty. If the condition is not met, the check is retried until the specified timeout is reached (if greater than zero). This action is supported for cloud storage backends.
  - `IMAP`. Enables integration with IMAP mailboxes. This feature allows you to automatically fetch email attachments from IMAP mailboxes and make them available within SFTPGo, either inside a user’s home directory or mapped into a virtual folder. Attachments can be periodically synchronized, enabling seamless ingestion of files delivered via email.
  - `ICAP`. Enables integration with ICAP servers to perform antivirus scanning and DLP checks as part of SFTPGo rules. After a file upload, the file can be streamed to an ICAP server for inspection. Depending on the scan result, different actions can be performed automatically, such as deleting the original file, moving it to a quarantine directory or virtual folder, or replacing it with the modified content returned by the ICAP server.

In actions, you can hard-code values such as file paths or email addresses. While this may work in some cases, it's generally better to use dynamic values that adapt to the specific context of the action. This is where dynamic placeholders come in. Placeholders allow you to insert values that are automatically replaced at runtime. They follow the format `{{.FieldName}}` and enable your actions to be more flexible and reusable.

We use Go’s template system under the hood. For a comprehensive overview, please refer to the official Go [template documentation](https://pkg.go.dev/text/template){:target="_blank"}. If you need advanced logic, conditions and loops are supported.

### Placeholders

- `{{.Name}}`. Username, virtual folder name, admin username for provider events, domain name for TLS certificate events. Format: string.
- `{{.ExtName}}`: External username, set to the email address used for authenticating public shares configured with email authentication. Format: string.
- `{{.Event}}`. Event name, for example `upload`, `download` for filesystem events or `add`, `update` for provider events. Format: string.
- `{{.Status}}`. Status for filesystem events. 1 means no error, 2 means a generic error occurred, 3 means quota exceeded error. Format: integer.
- `{{.Errors}}`. Error details. Format: list of strings.
- `{{.VirtualPath}}`. Path seen by SFTPGo users, for example `/adir/afile.txt`. Format: string.
- `{{.FsPath}}`. Full filesystem path, for example `/user/homedir/adir/afile.txt` or `C:/data/user/homedir/adir/afile.txt` on Windows. Format: string.
- `{{.VirtualTargetPath}}`. Virtual target path for rename and copy operations. Format: string.
- `{{.FsTargetPath}}`. Full filesystem target path for rename and copy operations. Format: string.
- `{{.ObjectName}}`. File/directory name, for example `afile.txt`, or provider object name. For data retention actions, this represents the username of the affected user. Format: string.
- `{{.ObjectType}}`. Object type for provider events: `user`, `group`, `admin` and so on. Format: string.
- `{{.FileSize}}`. File size. Format: int64.
- `{{.Elapsed}}`. Elapsed time as milliseconds for filesystem events. Format: int64.
- `{{.Protocol}}`. Used protocol, for example `SFTP`, `FTP`. Format: string.
- `{{.IP}}`. Client IP address. Format: string.
- `{{.Role}}`. User or admin role. Format: string.
- `{{.Email}}`. For filesystem events, this is the email associated with the user performing the action. For the provider events, this is the email associated with the affected user or admin. Blank in all other cases. Format: string.
- `{{.Timestamp}}`. Event timestamp. Format: time object. A time object has several useful methods: `UTC`, `Local`, `Unix`, `UnixMilli`, `Year`, `Month`, `Day`, `Hour`, `Minute`, and `Second`.
- `{{.UID}}`. Unique ID. Format: string.
- `{{.Object}}`. Provider object data with sensitive fields removed. It’s an object that includes a `JSON` method to get its JSON representation. For example, use `"ObjectJSON": {{.Object.JSON}}`. If you need the JSON as a string, use `"ObjectString": {{.Object.JSON | toJson}}`. This placeholder also allow to access some inner objects:
  - `Share`, if the event is related to a share. For example `{{.Object.Share.ExpiresAt}}` or `{{.Object.Share.Scope}`.
  - `User`, if the event is related to a user.
  - `Admin`, if the event is related to an admin.
  - `Group`, if the event is related to a group.
- `{{.RetentionReports}}`. Data retention reports as zip compressed CSV files. Supported as email attachment, file path for multipart HTTP request and as single parameter for HTTP requests body. Data retention reports contain details on the number of files deleted and the total size deleted for each folder. Format: string
- `{{.IDPFields}}`. Custom fields from the Identity Provider. Format: object. The structure depends on the specific custom claims configured in your Identity Provider.
- `{{.Metadata}}`. Cloud storage metadata represented as key/value pairs, where both keys and values are strings. You can use `range` to iterate over the keys and values.

The `{{.Timestamp}}` time object provides several useful methods:

- `UTC`: returns the time in UTC.
- `Local`: returns the time in the local timezone.
- `Unix`: returns the Unix timestamp in seconds.
- `UnixMilli`: returns the Unix timestamp in milliseconds.
- `Year`, `Month`, `Day`: return the date components.
- `Hour`, `Minute`, `Second`: return the time components.
- `Format(layout string)`: formats a time object using Go's layout reference `2006-01-02 15:04:05`.

Some examples:

- `{{ .Timestamp.Unix }}` returns the Unix timestamp in seconds.
- If `{{.Timestamp}}` is set to July 4, 2025 at 14:30, `{{ .Timestamp.Format "2006-01-02" }}` outputs `2025-07-04`, `{{ .Timestamp.Format "02/01/2006 15:04" }}` outputs `04/07/2025 14:30`, and `{{ .Timestamp.Format "Monday, 02 Jan 2006 at 15:04" }}` outputs `Friday, 04 Jul 2025 at 14:30`.

### Helper functions

You can use SFTPGo specific helper functions to transform or format the placeholder values. These functions allow you to do things like:

- Convert data to JSON: `{{ toJson .VirtualPath }}`.
- Format datetime: `{{ .Timestamp.UTC.Format "2006-01-02T15:04:05.000" }}`.
- Get the parent directory for a path: `{{ pathDir .VirtualPath }}`.

They can be used in either of the following forms: `{{ toJson .VirtualPath }}` or `{{ .VirtualPath | toJson }}`.

The first form calls the `toJson` function directly with `.VirtualPath` as its argument. The second form uses the pipe (`|`) operator, which passes the value on its left (`.VirtualPath`) as the input to the function on its right (`toJson`).

The pipe syntax is particularly useful when chaining multiple functions together, allowing you to transform data step-by-step in a clear and readable way.

Supported built-in functions:

- `toJson` converts any value to its JSON representation; since `.VirtualPath` is a string, `{ "path": {{ toJson .VirtualPath }} }` outputs `{ "path": "/mydir/myfile.txt" }`, and since `.Metadata` is a map of strings, `{{ toJson .Metadata }}` outputs `{"author":"alice","version":"1.0"}`. Using `toJson` ensures that strings are always correctly quoted and special characters properly escaped for safe inclusion in JSON.
- `toJsonUnquoted` works like `toJson`, but if the input is a string, it returns the JSON value without the surrounding quotes; for other types it behaves like  `toJson`. This is useful when you want to concatenate a dynamic JSON string with fixed text without extra quotes. Example: `{ "out_dir": "/basedir/{{ toJsonUnquoted .ObjectName }}" }` outputs `{"out_dir": "/basedir/myfile.txt"}`.
- `toBase64` converts a string value to its Base64 representation.
- `toHex` converts a string value to its hexadecimal representation.
- `urlEscape` encodes a string for safe use in query parameters Example: `{{ urlEscape .Email }}` outputs `user%40example.com`).
- `urlPathEscape` encodes a string for safe use in URL paths. Exammple: `{{ urlPathEscape .VirtualPath }}` outputs: `folder%20name%2Ffile.txt`.
- `pathDir` returns the directory part of a path. Example: if `.VirtualPath` is `/a/b/file.txt`, `{{ pathDir .VirtualPath }}` outputs `/a/b`.
- `pathBase` returns the last element of a path. Example: if `.VirtualPath` is `/a/b/file.txt`, `{{ pathBase .VirtualPath }}` outputs `file.txt`.
- `pathExt` returns the file extension. Example: if `.VirtualPath` is `/a/b/file.txt`, `{{ pathExt .VirtualPath }}` outputs `.txt`.
- `pathJoin` joins multiple path segments into a clean path. Example: `{{ pathJoin (stringSlice "/a" .VirtualPath "final") }}` with `.VirtualPath` as `b/c` outputs `/a/b/c/final`.
- `filePathJoin` joins multiple elements into a clean filesystem path using the correct separator for the OS. It’s similar to pathJoin, but `filePathJoin` should be used for real filesystem paths like `.FsPath`, while `pathJoin` is for virtual paths like `.VirtualPath`.
- `stringSlice` creates a list of strings. Example: `{{ pathJoin (stringSlice "/a" .VirtualPath "final") }}` with `.VirtualPath` as `b/c` outputs `/a/b/c/final`; it's useful when you need to pass multiple strings as a slice to functions like `pathJoin` or `filePathJoin`.
- `stringJoin` joins a list of strings into one string with a specified separator. Example: `{{ stringJoin .Errors ", " }}`.
- `stringTrimSuffix` removes a specified suffix from a string if present. Example: `{{ stringTrimSuffix .VirtualPath ".jpg" }}`.
- `stringTrimPrefix` removes a specified prefix from a string if present.
- `stringReplace` replaces all occurrences of a substring with another string. Example: `{{ stringReplace .VirtualPath "/dir1" "/dir2" }}`.
- `stringHasPrefix` checks if a string starts with a specified prefix. Example: `{{- if stringHasPrefix .VirtualPath "/dir2" -}}found{{- end -}}`.
- `stringHasSuffix` checks if a string ends with a specified suffix.
- `stringToLower` converts a string to lowercase. Example: `{{ stringToLower .VirtualPath }}`.
- `stringToUpper` converts a string to uppercase.
- `createDict` builds a map from alternating key-value pairs. Example: `{{- $statusMap := createDict 1 "OK" 2 "KO" -}}` creates a map where 1 maps to "OK" and 2 maps to "KO".
- `mapToString` looks up a value in a map by a given key. Example: `{{ (mapToString .Status $statusMap) | toJson }}` returns the string mapped to `.Status` in $statusMap, encoded as JSON.
- `humanizeBytes` converts a numeric byte value into a human-readable string with appropriate units (e.g., KB, MB, GB). It formats the input size by scaling it down and appending the correct unit suffix to improve readability. For example, an input of 10000 bytes is rendered as 10 KB. Example: `{{ humanizeBytes .FileSize }}`.
- `fromMillis` converts a Unix timestamp expressed in milliseconds into time object. Example: `(fromMillis $admin.CreatedAt).Format "2006-01-02 15:04:05"`

Some more examples.

This example shows how to build a JSON object, such as the body of an HTTP request, using advanced template features.

```json
{{- $statusMap := createDict 1 "OK" 2 "KO" -}}
{
  "Name": {{.Name | toJson}},
  "VirtualPath": {{.VirtualPath | toJson}},
  "Status": {{.Status}},
  "StatusString": {{ (mapToString .Status $statusMap) | toJson }},
  "Metadata": {{.Metadata | toJson}},
  "ObjectString": {{.Object.JSON | toJson}},
  "ObjectJSON": {{.Object.JSON}}
}
```

First, a map `$statusMap` is created with `createDict` to translate status codes (1 and 2) into strings ("OK" and "KO").

The JSON object includes several fields from the current context:

- "Name" and "VirtualPath" are converted to JSON strings with `toJson`.
- "Status" outputs the raw status code.
- "StatusString" uses `mapToString` to get the human-readable status from `$statusMap`, then converts it to a JSON string.
- "Metadata" outputs metadata as JSON.
- "ObjectString" outputs the JSON representation of `.Object` as a JSON string.
- "ObjectJSON" outputs the raw JSON object from `.Object.JSON`.

This approach allows mixing raw values, JSON strings, and mapped values seamlessly in a structured JSON output.

In Go templates, the `-` inside `{{-` or `-}}` trims whitespace immediately before or after the template tag. For example, `{{-` removes any whitespace to the left of the tag, and `-}}` removes whitespace to the right. This helps keep the generated output clean by avoiding unwanted spaces or newlines.

Another example:

```json
{{- $keyPrefix := stringJoin (stringSlice "users" .Name) "/" -}}
{
  "username": {{toJson .Name}},
  "status": 1,
  "permissions": {"/":["*"]},
  "filesystem": {
    "provider": 1,
    "s3config": {
      "bucket": "default",
      "region": "default",
      "key_prefix": {{ $keyPrefix | toJson }}
    }
  },
  "groups": [
    {{- $roles := .IDPFields.sftpgo_role -}}
    {{- range $i, $role := $roles -}}
        {{- if ne $i 0}},{{end}}
      {"type": {{if eq $i 0}}1{{else}}2{{end}},
      "name": {{$role | toJson}}}
    {{- end}}
    ]
}
```

This example builds a JSON object (for example, a user configuration) using advanced templating techniques:

- `$keyPrefix` is created by joining "users" and `.Name` with a slash (`/`), e.g., if `.Name` is "alice", `$keyPrefix` becomes "users/alice".
- The JSON object includes fixed fields like "username" (JSON-encoded `.Name`), "status", "permissions", and "filesystem" settings.
- Inside "filesystem", the "key_prefix" is set to the value of `$keyPrefix`.
- The "groups" array is populated from the `.IDPFields.sftpgo_role` claim, which is a list of roles (e.g., `["group1", "group2", "group3"]`). Using range, it loops over the roles: the first role gets `"type": 1` (primary group), the others `"type": 2` (secondary groups). Each group object includes "name" set to the role name, properly JSON-encoded.

This template dynamically generates user-related JSON data by combining static values, computed fields, and information from identity provider claims. It can be used, for example, to automatically create SFTPGo users after a successful Identity Provider login.

### Virtual folders

Virtual folders can be combined with filesystem actions. You can define:

- The source folder. Actions triggered by filesystem events, such as uploads or downloads, use the filesystem associated with the user. By specifying a source folder, you can control which filesystem is used. This is especially useful for events that aren't tied to a user, such as scheduled tasks and advanced workflows.
- The target folder. By specifying a target folder, you can use a different filesystem for target paths than the one associated with the user who triggered the action. This is useful for moving files to another storage backend, such as a different S3 bucket or an external SFTP server, accessing restricted areas of the same storage backend, supporting scheduled actions, or enabling more advanced workflows.

### Migration from Previous Versions or the Open-Source Edition

Starting with version `v2.7.20250726`, SFTPGo introduced a new, more powerful templating system for the EventManager.

If you're upgrading from a version **prior to `v2.7.20250726`** or from the Open-Source version, you will need to **manually migrate your existing actions** to the new templating syntax to ensure correct behavior.

In earlier versions (both Enterprise and open-source), event actions relied on simple placeholder replacement and attempted to automatically determine the output format, such as plain text or JSON, based on headers like `Content-Type`. This approach was limited and sometimes unreliable. Now, you need to be more explicit by using `toJson` and related functions where appropriate to ensure correct formatting.

Additionally, some placeholders were removed because their functionality can now be easily achieved using the built-in functions.

Removed placeholders:

- `{{.StatusString}}`. Instead, create a template variable using `createDict`, for example: `{{- $statusMap := createDict 1 "OK" 2 "KO" 3 "Quota exceeded" -}}`. Then retrieve the status string with: `{{ mapToString .Status $statusMap }}`. Bonus: this approach lets you customize the status messages easily.
- `{{.ErrorString}}`. Instead, use the `{{.Errors}}` placeholder, which contains the list of errors, and join them into a string with: `{{ stringJoin .Errors ", " }}` to get the same output as the old placeholder.
- `{{.EscapedVirtualPath}}`. Instead, use `{{ urlEscape .VirtualPath }}`. Bonus: you can apply `urlEscape` to any placeholder without needing separate escaped variants for each one.
- `{{.VirtualDirPath}}`, `{{.VirtualTargetDirPath}}`. Instead, use `{{ pathDir .VirtualPath }}` and `{{ pathDir .VirtualTargetPath }}`.
- `{{.Ext}}`. Instead use `{{ pathExt .VirtualPath }}`. Bonus: you can apply `pathExt` to any placeholder containing a path.
- `{{.TargetName}}`. Instead use `{{ pathBase VirtualTargetPath }}`.
- `{{.DateTime}}`, `{{.Year}}`, `{{.Month}}`, `{{.Day}}`, `{{.Hour}}`, `{{.Minute}}`: these individual placeholders are replaced by a single `{{.Timestamp}}` time object. You can format it as needed, for example:
`{{ .Timestamp.UTC.Fomat "2006-01-02T15:04:05.000" }}`. Additionally, you can call any method supported by the Go time object on `.Timestamp`, such as `UTC`, `Local`, `Unix`, `UnixMilli`, `Year`, `Month`, `Day`, `Hour`, `Minute`, and `Second`.
- `{{.ObjectData}}` and `{{.ObjectDataString}}`. These placeholders have been replaced by the `{{.Object}}` placeholder. Use `{{.Object.JSON}}` to get the equivalent of the old `{{.ObjectData}}`, and `{{.Object.JSON | toJson}}` to get the equivalent of `{{.ObjectDataString}}`.
- `{{.Metadata}}`, `{{.MetadataString}}`. The `{{.Metadata}}` placeholder is now an object. Use `{{ toJson .Metadata }}` to get the equivalent of the old `{{.Metadata}}`, and `{{toJson .Metadata | toJson}}` to get the equivalent of `{{.MetadataString}}`.
- `{{.IDPField<fieldname>}}`. These individual placeholders have been replaced by the generic `{{.IDPFields}}` object. You can now access fields using `{{ .IDPFields.fieldname }}`. Bonus: Previously, only string fields were available; now all fields are propagated with their original types as defined in your Identity Provider.

If you need assistance migrating your actions, please don’t hesitate to contact us.
