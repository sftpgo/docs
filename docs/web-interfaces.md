# Web UIs

## WebAdmin

The WebAdmin UI allows to easily create and manage your users, folders, groups and other resources.
Groups simplify the administration of multiple accounts by letting you assign settings once to a group, instead of multiple times to each individual user.

With the default `httpd` configuration, the web admin is available at the following URL:

[http://127.0.0.1:8080/web/admin](http://127.0.0.1:8080/web/admin){:target="_blank"}

If no admin user is found within the data provider, typically after the initial installation, SFTPGo will ask you to create the first admin. You can also pre-create an admin user by loading initial data or by configuration/env vars.

The web interface can be configured over HTTPS and to require mutual TLS authentication in addition to administrator credentials.

## WebClient

The SFTPGo WebClient allows end users to change their credentials, browse and manage their files in the browser and setup two-factor authentication which works with Authy, Google Authenticator and other compatible apps.

From the WebClient each authorized user can also create HTTP/S links to externally share files and folders securely, by setting limits to the number of downloads/uploads, protecting the share with a password, limiting access by source IP address, setting an automatic expiration date.

The web interface can be globally disabled within the `httpd` configuration via the `enable_web_client` key or on a per-user basis by adding `HTTP` to the denied protocols.
Public keys management can be disabled, per-user, using a specific permission.
The WebClient allows you to download multiple files or folders as a single zip file, any non regular files (for example symlinks) will be silently ignored.

With the default `httpd` configuration, the WebClient is available at the following URL:

[http://127.0.0.1:8080/web/client](http://127.0.0.1:8080/web/client){:target="_blank"}

## Internationalization

SFTPGo uses the [i18next](https://www.i18next.com/){:target="_blank"} framework for managing translating phrases in WebAdmin and WebClient.

Support for internationalization is experimental and may be removed in future releases. It mainly depends on the maintenance effort required and how useful this feature is for SFTPGo users. We currently only support English and Italian.

The translations are available via [Crowdin](https://crowdin.com/project/sftpgo){:target="_blank"}, who have granted us an open source license.

Translating phrases is hard, even lexically correct translations in the wrong context can completely change the meaning and confuse users.
Also, this is an ongoing effort, translations will be added, removed, updated in every release, even minor ones.

If you want to add support for a new language we require:

- you are a native speaker of the target language with the technical skills necessary to correctly contextualize SFTPGo translation phrases and/or a professional translator
- you/your company ensure long-term commitment to the project and help the project to be long-term sustainable

We will also try to merge translations that are almost complete and constantly updated, but we have to manually check that they do not contain offensive or inappropriate words, so especially the first time we merge them, it will take a lot of time and work, so we cannot guarantee that all contributed localizations will be merged.
