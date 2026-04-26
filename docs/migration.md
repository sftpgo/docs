---
description: "Migrate SFTPGo Event Manager rules to the new Go template syntax introduced in v2.7.20250726."
---

# Migration from Previous Versions

Starting with version `v2.7.20250726`, SFTPGo introduced a new, more powerful templating system for the Event Manager.

If you are upgrading from a version **prior to `v2.7.20250726`** or from the **open-source edition**, you need to manually migrate your existing actions to the new templating syntax.

## What changed

In earlier versions, event actions relied on simple placeholder replacement and attempted to automatically determine the output format (plain text or JSON) based on headers like `Content-Type`. This approach was limited and sometimes unreliable.

The new system uses Go's full [template engine](https://pkg.go.dev/text/template){:target="_blank"}, giving you explicit control over formatting with functions like `toJson`, `stringJoin`, and `pathDir`. You must now be explicit about how values are formatted — for example, using `toJson` to ensure correct quoting and escaping in JSON output.

See the [Placeholders & Templates](placeholders.md) reference for the complete list of available placeholders and helper functions.

## Removed placeholders

The following placeholders have been removed. Each entry shows the old syntax and the recommended replacement.

### `{{.StatusString}}`

Create a lookup map and use `mapToString`:

```json
{{- $statusMap := createDict 1 "OK" 2 "KO" 3 "Quota exceeded" -}}
{{ mapToString .Status $statusMap }}
```

This approach also lets you customize the status labels.

### `{{.ErrorString}}`

Use `{{.Errors}}` (a list) with `stringJoin`:

```json
{{ stringJoin .Errors ", " }}
```

### `{{.EscapedVirtualPath}}`

Use `urlEscape` — it works on any placeholder:

```json
{{ urlEscape .VirtualPath }}
```

### `{{.VirtualDirPath}}` and `{{.VirtualTargetDirPath}}`

Use `pathDir`:

```json
{{ pathDir .VirtualPath }}
{{ pathDir .VirtualTargetPath }}
```

### `{{.Ext}}`

Use `pathExt` — it works on any path:

```json
{{ pathExt .VirtualPath }}
```

### `{{.TargetName}}`

Use `pathBase`:

```json
{{ pathBase .VirtualTargetPath }}
```

### `{{.DateTime}}`, `{{.Year}}`, `{{.Month}}`, `{{.Day}}`, `{{.Hour}}`, `{{.Minute}}`

All replaced by the single `{{.Timestamp}}` time object. Format it as needed:

```json
{{ .Timestamp.UTC.Format "2006-01-02T15:04:05.000" }}
{{ .Timestamp.Year }}
{{ .Timestamp.Hour }}
```

See the [Timestamp section](placeholders.md#timestamp) for all available methods.

### `{{.ObjectData}}` and `{{.ObjectDataString}}`

Replaced by `{{.Object}}`:

- `{{.Object.JSON}}` — equivalent to the old `{{.ObjectData}}`
- `{{.Object.JSON | toJson}}` — equivalent to the old `{{.ObjectDataString}}`

### `{{.Metadata}}` and `{{.MetadataString}}`

`{{.Metadata}}` is now an object (map of strings):

- `{{ toJson .Metadata }}` — equivalent to the old `{{.Metadata}}`
- `{{ toJson .Metadata | toJson }}` — equivalent to the old `{{.MetadataString}}`

### `{{.IDPField<fieldname>}}`

Individual field placeholders have been replaced by the generic `{{.IDPFields}}` object:

```json
{{ .IDPFields.fieldname }}
```

Previously only string fields were available. Now all fields are propagated with their original types as defined in your Identity Provider.

## Need help?

If you need assistance migrating your actions, please don't hesitate to [contact us](https://sftpgo.com){:target="_blank"}.
