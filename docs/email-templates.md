---
description: "Customize SFTPGo email templates for password reset, password expiration, and share access code notifications."
---

# Email Templates

SFTPGo sends emails for specific system events such as password resets, password expiration notifications, and one-time access codes for shares. You can customize the subject and HTML body of these emails from the **Email Templates** section in the WebAdmin global configuration page.

When a custom template is configured in the database, it overrides the built-in default template. If left empty, the default template is used. Changes are applied across all instances in a cluster setup without requiring a service restart.

## Available Templates

### Password Reset

Sent when a user or admin requests a password reset via the "Forgot Password" flow.

**Available placeholders:**

| Placeholder | Description |
| ------------- | ------------- |
| `{{.Username}}` | The username of the account requesting the reset |
| `{{.Code}}` | The verification code (valid for 10 minutes) |

### Password Expiration

Sent as a notification when a user's password is about to expire or has already expired. Triggered by the password expiration check scheduled event action.

**Available placeholders:**

| Placeholder | Description |
| ------------- | ------------- |
| `{{.Username}}` | The username of the account |
| `{{.Days}}` | Number of days until password expiration. `0` or negative means the password has already expired |

### One-Time Access Code

Sent when a one-time access code is required, for example when accessing a share that requires email authentication.

**Available placeholders:**

| Placeholder | Description |
| ------------- | ------------- |
| `{{.Code}}` | The one-time access code |
| `{{.Email}}` | The email address of the recipient (when available) |
| `{{.ShareName}}` | The name of the share (when available) |
| `{{.ShareUsername}}` | The username of the share owner (when available) |
| `{{.ShareDesc}}` | The description of the share (when available) |

Not all placeholders are always populated. The `{{.Code}}` placeholder is always available. The share-specific placeholders (`{{.Email}}`, `{{.ShareName}}`, etc.) are populated only when the email is sent for a share access request.

## Template Syntax

Templates use [Go template syntax](https://pkg.go.dev/text/template). The most common usage is simply inserting placeholder values:

```html
<p>Your verification code is: <strong>{{.Code}}</strong></p>
```

You can also use conditional logic:

```html
{{if le .Days 0}}
<p>Your password has expired.</p>
{{else}}
<p>Your password expires in {{.Days}} days.</p>
{{end}}
```

## Email Subjects

Each template also has a configurable **Subject** field. Subjects support the same placeholder syntax as the body — you can use Go template variables to create dynamic subjects. For example:

```text
Reset code for {{.Username}}
```

If the subject is left empty, the default subject is used. Subjects configured in the UI take precedence over the `SFTPGO_HOOK__*_EMAIL_SUBJECT` environment variables. Note that environment variable subjects do not support placeholders.

| Template | Default Subject | Environment Variable (fallback) |
| ---------- | ---------------- | -------------------------------- |
| Password Reset | `Email Verification Code for {{.Username}}` | `SFTPGO_HOOK__PASSWORD_FORGOT_EMAIL_SUBJECT` |
| Password Expiration | `SFTPGo password expiration notification` | `SFTPGO_HOOK__PASSWORD_EXPIRATION_EMAIL_SUBJECT` |
| One-Time Access Code | `One-time access code` | `SFTPGO_HOOK__SHARE_CODE_EMAIL_SUBJECT` |

## HTML Editor

The WebAdmin UI provides a visual HTML editor for composing email templates. The editor supports:

- Rich text formatting (bold, italic, lists, etc.)
- Image embedding via Base64 or external links
- Table creation and editing

## Backward Compatibility

If no custom templates are configured in the database, SFTPGo uses the built-in default templates from the `templates/email` directory. The `smtp.templates_path` configuration option for overriding templates on disk continues to work as a fallback.

The precedence order is:

1. Custom template from database (configured via WebAdmin)
2. Template from disk (via `smtp.templates_path` configuration)
3. Built-in default template
