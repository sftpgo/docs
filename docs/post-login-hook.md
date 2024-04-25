# Post-login hook

This hook is executed after a login or after closing a connection for authentication timeout. Defining an appropriate `post_login_scope` you can get notifications for failed logins, successful logins or both.

The `post-login-hook` can be defined as the absolute path of your program or an HTTP URL.

If the hook defines an external program it can reads the following environment variables:

- `SFTPGO_LOGIND_USER`, it contains the user serialized as JSON. The username is empty if the connection is closed for authentication timeout
- `SFTPGO_LOGIND_IP`
- `SFTPGO_LOGIND_METHOD`, possible values are `publickey`, `password`, `keyboard-interactive`, `publickey+password`, `publickey+keyboard-interactive`, `TLSCertificate`, `TLSCertificate+password` or `no_auth_tried`, `IDP` (external identity provider)
- `SFTPGO_LOGIND_STATUS`, 1 means login OK, 0 login KO
- `SFTPGO_LOGIND_PROTOCOL`, possible values are `SSH`, `FTP`, `DAV`, `HTTP`, `OIDC` (OpenID Connect)

Global environment variables are cleared, for security reasons, when the script is called. You can set additional environment variables in the "command" configuration section.
The program must finish within 20 seconds.

If the hook is an HTTP URL then it will be invoked as HTTP POST. The login method, the used protocol, the ip address and the status of the user are added to the query string, for example `<http_url>?login_method=password&ip=1.2.3.4&protocol=SSH&status=1`.
The request body will contain the user serialized as JSON.

The structure for SFTPGo users can be found within the [OpenAPI schema](https://github.com/drakkan/sftpgo/blob/2.6.x/openapi/openapi.yaml){:target="_blank"}.

The HTTP hook will use the global configuration for HTTP clients and will respect the retry, TLS and headers configurations. See the HTTP Clients (`http`) section of the [config reference](config-file.md#http-clients).

The `post_login_scope` supports the following configuration values:

- `0` means notify both failed and successful logins
- `1` means notify failed logins. Connections closed for authentication timeout are notified as failed logins. You will get an empty username in this case
- `2` means notify successful logins
