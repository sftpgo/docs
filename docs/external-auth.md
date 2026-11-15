---
description: "Authenticate open-source SFTPGo users via external programs or HTTP endpoints. Integrate with custom databases or any external identity source."
---

# External Authentication

To enable external authentication, you must set the absolute path of your authentication program or an HTTP URL using the `external_auth_hook` key in your configuration file.

The external program can read the following environment variables to get info about the user trying to authenticate:

- `SFTPGO_AUTHD_USERNAME`
- `SFTPGO_AUTHD_USER`, STPGo user serialized as JSON, empty if the user does not exist within the data provider
- `SFTPGO_AUTHD_IP`
- `SFTPGO_AUTHD_PROTOCOL`, possible values are `SSH`, `FTP`, `DAV`, `HTTP`
- `SFTPGO_AUTHD_PASSWORD`, not empty for password authentication
- `SFTPGO_AUTHD_PUBLIC_KEY`, not empty for public key authentication
- `SFTPGO_AUTHD_KEYBOARD_INTERACTIVE`, set to `1` for keyboard interactive authentication. This request carries no credential, see [keyboard interactive authentication](#keyboard-interactive-authentication)
- `SFTPGO_AUTHD_TLS_CERT`, TLS client certificate PEM encoded. Not empty for TLS certificate authentication

Global environment variables are cleared, for security reasons, when the script is called. You can set additional environment variables in the "command" configuration section.
The program can inspect the SFTPGo user, if it exists, using the `SFTPGO_AUTHD_USER` environment variable.
The program must write, on its standard output:

- a valid SFTPGo user serialized as JSON if the authentication succeeds. The user will be added/updated within the defined data provider
- an empty string, or no response at all, if authentication succeeds and the existing SFTPGo user does not need to be updated. For a password login SFTPGo stores the password the hook has just accepted; for a public key login the key must already be stored for the user.
- a user with an empty username if the authentication fails

If the hook is an HTTP URL then it will be invoked as HTTP POST. The request body will contain a JSON serialized struct with the following fields:

- `username`
- `ip`
- `user`, STPGo user, omitted if the user does not exist within the data provider
- `protocol`, possible values are `SSH`, `FTP`, `DAV`, `HTTP`
- `password`, not empty for password authentication
- `public_key`, not empty for public key authentication
- `keyboard_interactive`, set to `1` for keyboard interactive authentication. This request carries no credential, see [keyboard interactive authentication](#keyboard-interactive-authentication)
- `tls_cert`, TLS client certificate PEM encoded. Not empty for TLS certificate authentication

If authentication succeeds the HTTP response code must be 200 and the response body can be:

- a valid SFTPGo user serialized as JSON. The user will be added/updated within the defined data provider
- empty, the existing SFTPGo user does not need to be updated, as for an empty program response.

If the authentication fails the HTTP response code must be != 200 or the returned SFTPGo user must have an empty username.

If the hook returns a user who is only allowed to authenticate using public key + password (multi step authentication), your hook will be invoked for each authentication step, so it must validate the public key and password separately. SFTPGo will take care that the client uses the allowed sequence.

Actions defined for users added/updated will not be executed in this case and an already logged in user with the same username will not be disconnected.

The program hook must finish within 30 seconds, the HTTP hook timeout will use the global configuration for HTTP clients.

This method is slower than built-in authentication, but it's very flexible as anyone can easily write his own authentication hooks.
You can also restrict the authentication scope for the hook using the `external_auth_scope` configuration key:

- `0` means all supported authentication scopes. The external hook will be used for password, public key, keyboard interactive and TLS certificate authentication
- `1` means passwords only
- `2` means public keys only
- `4` means keyboard interactive only
- `8` means TLS certificate only

You can combine the scopes. For example, 3 means password and public key, 5 means password and keyboard interactive, and so on.

With the default scope the hook receives every authentication method listed above and must answer each of them. A method outside the scope is verified by SFTPGo against the credentials stored for the user, without calling the hook: for example the last password the hook accepted, or the public keys of the user, including those the user adds from the WebClient.

Let's see a very basic example. Our sample authentication program will only accept user `test_user` with any password or public key.

```shell
#!/bin/sh

if test "$SFTPGO_AUTHD_USERNAME" = "test_user"; then
  echo '{"status":1,"username":"test_user","expiration_date":0,"home_dir":"/tmp/test_user","uid":0,"gid":0,"max_sessions":0,"quota_size":0,"quota_files":100000,"permissions":{"/":["*"],"/somedir":["list","download"]},"upload_bandwidth":0,"download_bandwidth":0,"filters":{"allowed_ip":[],"denied_ip":[]},"public_keys":[]}'
else
  echo '{"username":""}'
fi
```

The structure for SFTPGo users can be found within the [OpenAPI schema](https://github.com/drakkan/sftpgo/blob/main/openapi/openapi.yaml){:target="_blank"}.

You can instruct SFTPGo to cache the external user by setting an `external_auth_cache_time` in user object returned by your hook. The `external_auth_cache_time` defines the cache time in seconds.

You can disable the hook on a per-user basis so that you can mix external and internal users.

An example authentication program allowing to authenticate against an LDAP server can be found inside the source tree [ldapauth](https://github.com/drakkan/sftpgo/tree/main/examples/ldapauth){:target="_blank"} directory.

An example server, to use as HTTP authentication hook, allowing to authenticate against an LDAP server can be found inside the source tree [ldapauthserver](https://github.com/drakkan/sftpgo/tree/main/examples/ldapauthserver){:target="_blank"} directory.

If you have an external authentication hook that could be useful to others too, please let us know and/or please send a pull request.

## Keyboard interactive authentication

A keyboard interactive request carries the username only: the hook decides whether the account can use this method. A refused username receives no challenge, and the client can move on to password authentication.

SFTPGo verifies the challenges itself, not through this hook: the builtin flow checks the password against the one stored in SFTPGo, which is the last password the hook accepted, and then the one-time code if enabled. To verify the password against a directory on every login, use a [keyboard interactive hook](keyboard-interactive.md); it can also ask for the one-time code or leave its check to SFTPGo. When keyboard interactive authentication is not needed, set [`keyboard_interactive_authentication`](config-file.md#sshsftp-server) to `false`: clients then use password authentication and the hook receives the password. Over SSH the [second factor](tutorials/two-factor-authentication.md) is asked only within keyboard interactive authentication, so this setting suits installations that do not use it for SSH.

The hook is invoked for every username, including the ones that do not exist inside SFTPGo, and SFTPGo uses the username of the account it returns. To handle every unknown username in the same way, return the same disabled placeholder account, with `status` set to `0`, for all of them: a single account is created the first time and updated on the following logins, the same [challenges](keyboard-interactive.md#challenges-for-unknown-usernames) are sent as for an existing account, and every login of the placeholder is denied.

## Trust model

Hooks and plugins are trusted components: they run with the privileges of the SFTPGo service and they are the authority on what SFTPGo asks them. The response is applied as it is given: the account it names, and the settings it carries, are what gets stored and what the session runs as.

What a response causes therefore belongs to the integration that produced it. Answering for an account grants access to that account, exactly as an entry in the data provider does. Validate the credentials received before answering, and answer consistently when SFTPGo asks more than once for the same login, as it does when a login is verified in more than one step.
