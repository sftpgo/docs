---
description: "Dynamic placeholders and Go template syntax for SFTPGo Event Manager actions. Customize emails, HTTP calls, and commands with event data."
---

# Placeholders & Templates

Actions in the [Event Manager](eventmanager.md) can use dynamic **placeholders** — variables that are replaced at runtime with data from the triggering event. Placeholders follow the Go template syntax `{{.FieldName}}` and can be used in HTTP request bodies, email subjects and bodies, command environment variables, filesystem action paths, and more.

SFTPGo uses Go's [text/template](https://pkg.go.dev/text/template){:target="_blank"} engine under the hood. Conditions, loops, and variable assignments are fully supported.

## Available placeholders

### Core fields

| Placeholder | Type | Description |
| ------------- | ------ | ------------- |
| `{{.Name}}` | string | Username for filesystem events; virtual folder name, admin username, or domain name depending on the trigger type. |
| `{{.ExtName}}` | string | External username. Set to the email address used for authenticating public shares configured with email authentication. |
| `{{.Event}}` | string | Event name — e.g., `upload`, `download` for filesystem events, or `add`, `update`, `delete` for provider events. |
| `{{.Status}}` | integer | Status code for filesystem events: `1` = success, `2` = generic error, `3` = quota exceeded. |
| `{{.Errors}}` | list of strings | Error details, if any. Use `{{ stringJoin .Errors ", " }}` to format as a single string. |
| `{{.VirtualPath}}` | string | Path as seen by the user — e.g., `/documents/report.pdf`. |
| `{{.FsPath}}` | string | Full filesystem path — e.g., `/home/user/documents/report.pdf` or the equivalent cloud storage key. |
| `{{.VirtualTargetPath}}` | string | Virtual target path for rename and copy operations. |
| `{{.FsTargetPath}}` | string | Full filesystem target path for rename and copy operations. |
| `{{.ObjectName}}` | string | File or directory name (e.g., `report.pdf`), or the provider object name. For data retention actions: the affected username (user-scoped) or the source folder name (folder-scoped). |
| `{{.ObjectType}}` | string | Object type for provider events: `user`, `group`, `admin`, `folder`, `share`, etc. |
| `{{.FileSize}}` | int64 | File size in bytes. |
| `{{.Elapsed}}` | int64 | Elapsed time in milliseconds for filesystem events. |
| `{{.Protocol}}` | string | Protocol used — e.g., `SFTP`, `FTP`, `HTTP`, `WebDAV`. |
| `{{.IP}}` | string | Client IP address. |
| `{{.Role}}` | string | User or admin role. |
| `{{.Email}}` | string | For filesystem events: the email of the user performing the action. For provider events: the email of the affected user or admin. Blank in other cases. |
| `{{.UID}}` | string | Unique event identifier. |

### Timestamp

`{{.Timestamp}}` is a time object with the following methods:

| Method | Description | Example output |
| -------- | ------------- | ---------------- |
| `.UTC` | Time in UTC | — |
| `.Local` | Time in server's local timezone | — |
| `.Unix` | Unix timestamp in seconds | `1751644200` |
| `.UnixMilli` | Unix timestamp in milliseconds | `1751644200000` |
| `.Year`, `.Month`, `.Day` | Date components | `2025`, `July`, `4` |
| `.Hour`, `.Minute`, `.Second` | Time components | `14`, `30`, `0` |
| `.Format "layout"` | Custom format using Go's [reference time](https://pkg.go.dev/time#pkg-constants){:target="_blank"} `2006-01-02 15:04:05` | See examples below |

Format examples:

- `{{ .Timestamp.Format "2006-01-02" }}` outputs `2025-07-04`
- `{{ .Timestamp.Format "02/01/2006 15:04" }}` outputs `04/07/2025 14:30`
- `{{ .Timestamp.UTC.Format "2006-01-02T15:04:05.000" }}` outputs `2025-07-04T12:30:00.000`

### Object and Initiator

#### `{{.Object}}`

The provider object associated with the event, with sensitive fields removed. It exposes a `JSON` method and typed accessors for inner objects:

- `{{.Object.JSON}}` — JSON representation of the full object.
- `{{.Object.Share}}` — Share data, when the event involves a share. Example: `{{.Object.Share.ExpiresAt}}`.
- `{{.Object.User}}` — User data. Example: `{{.Object.User.Email}}`.
- `{{.Object.Admin}}` — Admin data. Example: `{{.Object.Admin.Description}}`.
- `{{.Object.Group}}` — Group data.

Field names match those documented in the [REST API](https://sftpgo.com/rest-api){:target="_blank"}, using PascalCase (e.g., `CreatedAt`, `Description`).

To embed the JSON in another JSON structure:

- As a nested JSON object: `"data": {{.Object.JSON}}`
- As a JSON string: `"data": {{.Object.JSON | toJson}}`

#### `{{.Initiator}}`

For provider events, exposes the entity (user or admin) that *initiated* the action — not the object being acted upon. For example, when an admin creates a user, `.Object` is the new user and `.Initiator` is the admin.

Resolution rules:

- **Share events**: the user who owns the share.
- **Other provider events** (user/admin/group creation, update, deletion): the admin who performed the action.
- **Self-edit operations**: the same entity as `.Object` (the user or admin editing their own profile).

The initiator is resolved **lazily** — SFTPGo only queries the database if the template actually references it.

Typed accessors:

- `{{.Initiator.User}}` — Initiator as a user. Example: `{{.Initiator.User.Email}}`.
- `{{.Initiator.Admin}}` — Initiator as an admin. Example: `{{.Initiator.Admin.Email}}`.
- `{{.Initiator.JSON}}` — JSON representation.

:warning: Calling the wrong accessor (e.g., `.Initiator.User` when the initiator is an admin) causes a template rendering error. Use conditional checks:

```json
{{if .Initiator.User}}User: {{.Initiator.User.Email}}{{end}}
{{if .Initiator.Admin}}Admin: {{.Initiator.Admin.Email}}{{end}}
```

### Collection placeholders

These placeholders are populated by specific action types and are available to subsequent actions in the same rule.

#### `{{.RetentionChecks}}`

Populated by the **Data retention check** action. List of retention check results — one item per user for user-scoped retention, a single item for folder-scoped retention. Each item contains:

| Field | Type | Description |
| ------- | ------ | ------------- |
| `Username` | string | User-scoped: the user whose files were checked. Folder-scoped: the canonical `__system__` identifier. |
| `Folder` | string | Source folder name. Populated only for folder-scoped retention; empty for user-scoped. |
| `Email` | list of strings | User's email addresses. Empty for folder-scoped retention. |
| `ActionName` | string | The name of the retention action. |
| `Type` | integer | `0` = delete, `1` = archive. |
| `DryRun` | boolean | `true` if the action ran in dry-run mode (no files deleted, no archive copies). |
| `Results` | list of objects | Per-folder results (see below). |

Each item in `Results`:

| Field | Type | Description |
| ------- | ------ | ------------- |
| `Path` | string | Folder path. |
| `Retention` | integer | Retention threshold in hours. |
| `DeletedFiles` | integer | Number of files removed. |
| `DeletedSize` | integer | Total bytes removed. |
| `Elapsed` | integer | Processing time in nanoseconds. |
| `Info` | string | Additional information. |
| `Error` | string | Error message, if any. |

`{{.RetentionReports}}` is also available as an email attachment or HTTP multipart file containing compressed CSV reports.

#### `{{.DryRun}}`

Top-level boolean populated by the **Data retention check** action. `true` only when **every** retention check recorded on the rule ran in dry-run mode — designed as a fail-safe gate so notifications cannot mistake a real deletion for a preview when a rule chains multiple retention actions. Useful for prefixing a subject with `[DRY RUN]` only when no real cleanup happened:

```text
{{ if .DryRun }}[DRY RUN] {{ end }}Retention report
```

When a rule chains multiple Data Retention actions with different DryRun states, `{{.DryRun}}` reports `false`. To act on each retention action's individual mode, iterate `{{range .RetentionChecks}}` and read the per-entry `.DryRun`.

#### `{{.ShareExpirationChecks}}`

Populated by the **Share expiration check** action. List of results grouped by user. Each item contains:

| Field | Type | Description |
| ------- | ------ | ------------- |
| `User` | object | The user who owns the shares. Fields match the [REST API](https://sftpgo.com/rest-api){:target="_blank"} in PascalCase. |
| `Results` | list of objects | Per-share results (see `{{.ShareExpirationResult}}` below). |

#### `{{.ShareExpirationResult}}`

Available **only** when **Split events** is enabled for the share expiration check. In this mode, each result triggers its own event, and standard placeholders like `{{.Name}}` and `{{.Email}}` are automatically set to the user associated with this specific result.

| Field | Type | Description |
| ------- | ------ | ------------- |
| `Share` | object | The full share object (PascalCase fields). Example: `{{.ShareExpirationResult.Share.Name}}`. |
| `Action` | integer | `1` = notify (advance warning), `2` = delete. |
| `Reason` | string | `max_tokens`, `expiration_date`, or `inactivity`. |
| `Expiration` | time object | The calculated expiration timestamp. |

#### `{{.EventReports}}`

Populated by the **Event report** action. See the [Event Report](event-report.md) documentation for the full field reference.

### Other placeholders

| Placeholder | Type | Description |
| ------------- | ------ | ------------- |
| `{{.IDPFields}}` | object | Custom fields from the Identity Provider. Structure depends on your IdP configuration. Example: `{{.IDPFields.sftpgo_role}}`. |
| `{{.Metadata}}` | map of strings | Cloud storage metadata (key/value pairs). Use `range` to iterate, or `{{ toJson .Metadata }}` for JSON output. |
| `{{.Shares}}` | lazy object | Shares associated with the file path of a filesystem event. Call `.Load` to retrieve them. Example: `{{ range .Shares.Load }}{{ range .Options.Emails }}{{ . }},{{ end }}{{ end }}`. |

## Go template built-ins

In addition to the SFTPGo-specific helpers documented below, every built-in function and action from Go's [text/template](https://pkg.go.dev/text/template){:target="_blank"} package is available.

### Formatting and printing

| Function | Description | Example |
| ---------- | ------------- | --------- |
| `printf` | Formats using Go's [fmt](https://pkg.go.dev/fmt){:target="_blank"} verbs and returns the result. | `{{ printf "%s uploaded %d bytes" .Name .FileSize }}` |
| `print` | Concatenates values using default formatting. | `{{ print .Name " at " .IP }}` |
| `println` | Like `print`, but adds a trailing newline. | `{{ println .VirtualPath }}` |

Common `printf` verbs:

- `%s` — string
- `%d` — integer (`%05d` zero-pads to width 5, e.g. `00042`)
- `%v` — default format for any value
- `%q` — Go-quoted string (escapes quotes and special characters)
- `%x` / `%X` — lowercase / uppercase hexadecimal
- `%.2f` — floating point with 2 decimal places

### Comparison and logic

| Function | Description |
| ---------- | ------------- |
| `eq`, `ne` | Equal, not equal. `eq` accepts multiple arguments: `eq a b c` is true if `a == b || a == c`. |
| `lt`, `le`, `gt`, `ge` | Less-than, less-or-equal, greater-than, greater-or-equal. |
| `and`, `or`, `not` | Logical operators. `and` / `or` short-circuit and return the first non-empty / non-zero argument. |

Example:

```
{{if and (eq .Event "upload") (gt .FileSize 1048576) }}
  Large file uploaded by {{.Name}}
{{end}}
```

### Collection operations

| Function | Description | Example |
| ---------- | ------------- | --------- |
| `len` | Length of a string, slice, map, or array. | `{{ len .Errors }}` |
| `index` | Returns the element at the given index or key. | `{{ index .Errors 0 }}`, `{{ index .IDPFields "sftpgo_role" }}` |
| `slice` | Slices a string, slice, or array. | `{{ slice .Name 0 3 }}` |

### Escaping

| Function | Description |
| ---------- | ------------- |
| `js` | Escapes for use inside a JavaScript string literal. |
| `html` | HTML-escapes `<`, `>`, `&`, and quotes. |

For URL escaping, use SFTPGo's `urlEscape` / `urlPathEscape` helpers (documented below).

### Control structures

Conditional blocks, loops, and variable assignment are template actions (not functions):

```
{{if eq .Status 1}}Success{{else}}Failed{{end}}

{{range .Errors}}
- {{.}}
{{end}}

{{range $i, $item := .RetentionChecks}}
Check {{$i}}: {{$item.Username}}
{{end}}

{{with .Object.User}}
User email: {{.Email}}
{{end}}

{{$label := "Upload"}}
{{$label}}: {{.VirtualPath}}
```

- `{{if}}` — conditional block; pair with `{{else}}` / `{{else if}}`.
- `{{range}}` — iterates over slices, arrays, or maps. `{{break}}` and `{{continue}}` are supported.
- `{{with}}` — narrows the `.` context for the block, skipping it if the value is empty.
- `$var := value` — declares a variable scoped to the surrounding block.
- `$var = value` — reassigns an existing variable.

For the complete reference, see the [text/template documentation](https://pkg.go.dev/text/template){:target="_blank"}.

## Helper functions

Helper functions transform or format placeholder values. They can be called in two equivalent ways:

- Direct: `{{ toJson .VirtualPath }}`
- Piped: `{{ .VirtualPath | toJson }}`

The pipe syntax is especially useful for chaining multiple transformations.

### Encoding and conversion

| Function | Description | Example |
| ---------- | ------------- | --------- |
| `toJson` | Converts any value to its JSON representation. Strings are quoted, special characters escaped. | `{{ toJson .VirtualPath }}` → `"/dir/file.txt"` |
| `toJsonUnquoted` | Like `toJson`, but strips surrounding quotes from string values. Other types behave like `toJson`. | `{{ toJsonUnquoted .ObjectName }}` → `file.txt` |
| `toBase64` | Encodes a string as Base64. | `{{ toBase64 .Name }}` |
| `toHex` | Encodes a string as hexadecimal. | `{{ toHex .Name }}` |

### URL encoding

| Function | Description | Example |
| ---------- | ------------- | --------- |
| `urlEscape` | Encodes a string for use in query parameters. | `{{ urlEscape .Email }}` → `user%40example.com` |
| `urlPathEscape` | Encodes a string for use in URL path segments. | `{{ urlPathEscape .VirtualPath }}` → `folder%20name%2Ffile.txt` |

### Path manipulation

| Function | Description | Example |
| ---------- | ------------- | --------- |
| `pathDir` | Returns the directory portion of a path. | `{{ pathDir "/a/b/file.txt" }}` → `/a/b` |
| `pathBase` | Returns the last element of a path. | `{{ pathBase "/a/b/file.txt" }}` → `file.txt` |
| `pathExt` | Returns the file extension. | `{{ pathExt "/a/b/file.txt" }}` → `.txt` |
| `pathJoin` | Joins path segments into a clean virtual path. Takes a string slice. | `{{ pathJoin (stringSlice "/a" .VirtualPath "final") }}` |
| `filePathJoin` | Like `pathJoin` but uses OS-specific separators. Use for `.FsPath` values. | `{{ filePathJoin (stringSlice "/data" .FsPath) }}` |

### String operations

| Function | Description | Example |
| ---------- | ------------- | --------- |
| `stringSlice` | Creates a list of strings. Useful as input for `pathJoin` or `stringJoin`. | `{{ stringSlice "a" "b" "c" }}` |
| `stringJoin` | Joins a list of strings with a separator. | `{{ stringJoin .Errors ", " }}` |
| `stringTrimSuffix` | Removes a suffix if present. | `{{ stringTrimSuffix .VirtualPath ".jpg" }}` |
| `stringTrimPrefix` | Removes a prefix if present. | `{{ stringTrimPrefix .VirtualPath "/data" }}` |
| `stringReplace` | Replaces all occurrences of a substring. | `{{ stringReplace .VirtualPath "/dir1" "/dir2" }}` |
| `stringHasPrefix` | Returns true if the string starts with the given prefix. | `{{if stringHasPrefix .VirtualPath "/dir"}}...{{end}}` |
| `stringHasSuffix` | Returns true if the string ends with the given suffix. | `{{if stringHasSuffix .VirtualPath ".csv"}}...{{end}}` |
| `stringContains` | Returns true if the string contains the given substring. | `{{if stringContains .VirtualPath "report"}}...{{end}}` |
| `stringToLower` | Converts to lowercase. | `{{ stringToLower .VirtualPath }}` |
| `stringToUpper` | Converts to uppercase. | `{{ stringToUpper .Name }}` |
| `slicesContains` | Returns true if a slice contains the given element. | `{{if slicesContains .Errors "timeout"}}...{{end}}` |

### Maps and formatting

| Function | Description | Example |
| ---------- | ------------- | --------- |
| `createDict` | Creates a map from alternating key-value pairs. | `{{ $m := createDict 1 "OK" 2 "KO" }}` |
| `mapToString` | Looks up a value in a map by key. | `{{ mapToString .Status $statusMap }}` |
| `humanizeBytes` | Formats a byte count as a human-readable string (KB, MB, GB, etc.). | `{{ humanizeBytes .FileSize }}` → `10 KB` |
| `fromMillis` | Converts a Unix timestamp in milliseconds to a time object. | `{{ (fromMillis $admin.CreatedAt).Format "2006-01-02" }}` |
| `fromNanos` | Converts a Unix timestamp in nanoseconds to a time object. | `{{ (fromNanos $event.Timestamp).Format "15:04:05" }}` |

## Template examples

### Building a JSON HTTP request body

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

- `Name` and `VirtualPath` are converted to JSON strings with `toJson`, ensuring proper quoting and escaping.
- `Status` is output as a raw integer.
- `StatusString` maps the integer to a human-readable label via `createDict` + `mapToString`, then quotes it.
- `Metadata` outputs the full metadata map as a JSON object.
- `ObjectString` is the object's JSON wrapped as a JSON string (double-encoded).
- `ObjectJSON` is the raw JSON object embedded directly.

:information_source: In Go templates, `{{-` and `-}}` trim whitespace before/after the tag. Use them to keep the output clean.

### Dynamic user creation from Identity Provider

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

This template generates a user configuration for the Identity Provider account check action:

- `$keyPrefix` is built by joining `"users"` and the username with `/` — e.g., `users/alice`.
- The `groups` array is populated from the `sftpgo_role` claim. The first role gets type `1` (primary group), subsequent roles get type `2` (secondary).
- All string values use `toJson` for safe JSON encoding.
