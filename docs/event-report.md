---
description: "Generate periodic event reports in SFTPGo: query filesystem activity, aggregate by user, and deliver as email attachments or via HTTP."
---

# Event Report

The Event Report action queries stored filesystem events and generates structured reports grouped by user. It is designed for **scheduled rules** that send periodic digest notifications — for example, a daily summary of all uploads and downloads.

This action requires the **event search plugin** to be installed and configured. Events are stored by the plugin and queried on demand when the action executes.

## How it works

1. A scheduled rule triggers the Event Report action.
2. The action queries the event search plugin for filesystem events within the configured time window.
3. Results are grouped by username and made available to subsequent actions in the same rule via the `{{.EventReports}}` placeholder.
4. Downstream actions (email, HTTP, command) can use the structured data to build notifications, dashboards, or audit logs.

## Configuration

### Filters

| Field | Description |
| ------- | ------------- |
| **Time window** | How far back to search, in minutes. For example, `30` means "events from the last 30 minutes". The default in the WebAdmin UI is 30. |
| **Filesystem actions** | Optional filter to include only specific event types (e.g., `upload`, `download`, `delete`, `rename`, `copy`, `mkdir`, `rmdir`, `ssh_cmd`). If empty, all actions are included. |
| **Statuses** | Optional filter by event outcome: `OK`, `error`, `quota exceeded`. If empty, all statuses are included. |

### Options

| Field | Description |
| ------- | ------------- |
| **Split reports** | When enabled, generates a separate event for each user instead of a single aggregated event. In split mode, `{{.Email}}` and `{{.ObjectName}}` are automatically set to the user's email and username, enabling per-user notifications. |

### Server-side limits

The maximum number of events returned per query is controlled by the environment variable `SFTPGO_HOOK__EVENT_REPORT_MAX_RESULTS` (default: `10000`). If this limit is reached, the `Truncated` field on the report is set to `true`, indicating that additional events may exist beyond what is included.

## Template data

When the Event Report action executes, it populates the `{{.EventReports}}` placeholder for downstream actions. This is a list of objects, one per user.

### Report fields

| Field | Type | Description |
| ------- | ------ | ------------- |
| `Username` | string | The user whose events are included. |
| `Email` | list of strings | The user's email addresses (primary + additional). |
| `Truncated` | boolean | `true` if the server-side result limit was reached. |
| `Events` | list of objects | The filesystem events (see below). |

### Event fields

Each item in the `Events` list represents a single filesystem event:

| Field | Type | Description |
| ------- | ------ | ------------- |
| `ID` | string | Unique event identifier. |
| `Timestamp` | int64 | Unix timestamp in nanoseconds. |
| `Action` | string | Event type: `upload`, `download`, `delete`, `rename`, `copy`, `mkdir`, `rmdir`, `ssh_cmd`. |
| `Username` | string | The user who performed the action. |
| `VirtualPath` | string | Path as seen by the user. |
| `VirtualTargetPath` | string | Target path for rename and copy operations. |
| `FileSize` | int64 | File size in bytes. |
| `Elapsed` | int64 | Duration in milliseconds. |
| `Status` | integer | `1` = success, `2` = error, `3` = quota exceeded. |
| `Protocol` | string | `SFTP`, `FTP`, `HTTP`, etc. |
| `IP` | string | Client IP address. |
| `SSHCmd` | string | The SSH command, populated only for `ssh_cmd` events. |

## Email attachment

When combining the Event Report with an email action, you can enable the **Attach event report** option on the email action. This automatically attaches the report as a **compressed CSV file** — one CSV per user, bundled in a ZIP archive. The attachment is generated only when report data is available.

This provides a convenient way to deliver detailed audit data to administrators without requiring them to access the SFTPGo UI.

## Usage examples

### Daily upload summary email

Create a scheduled rule that runs daily at 08:00 UTC:

1. **Event Report action**: Time window = `1440` (24 hours), Filesystem actions = `upload`, Split reports = disabled.
2. **Email action**: Enable "Attach event report". Subject: `Daily upload report — {{ .Timestamp.Format "2006-01-02" }}`. Body:

```text
Upload activity for the last 24 hours:

{{ range .EventReports -}}
User: {{ .Username }} — {{ len .Events }} event(s)
{{ end }}

See the attached CSV for details.
```

### Per-user weekly digest

Create a scheduled rule that runs every Monday at 07:00 UTC:

1. **Event Report action**: Time window = `10080` (7 days), Split reports = enabled.
2. **Email action**: Recipients = `{{ stringJoin .Email "," }}`, Subject = `Weekly activity report for {{ .ObjectName }}`. Body:

```text
Hi {{ .ObjectName }},

Here is your activity summary for the past week:

{{ range .EventReports -}}
{{ range .Events -}}
  {{ .Action }} — {{ .VirtualPath }} ({{ humanizeBytes .FileSize }}) — {{ (fromNanos .Timestamp).Format "2006-01-02 15:04" }}
{{ end -}}
{{ end }}
```

With split reports enabled, the rule fires once per user. `{{.Email}}` is automatically set to the user's email, and `{{.ObjectName}}` to the username.
