# REST API

SFTPGo supports REST API for pretty much everything, both for administrators and users.

REST API are protected using JSON Web Tokens (JWT) or API key authentication and can be served over HTTPS. You can also configure client certificate authentication in addition to JWT.

REST API can be disabled within the `httpd` configuration via the `enable_rest_api` key.

The OpenAPI 3 schema for the supported APIs can be found inside the source tree: [openapi.yaml](https://github.com/drakkan/sftpgo/blob/main/openapi/openapi.yaml "OpenAPI 3 specs"){:target="_blank"}. You can render the schema and try the API using the `/openapi` endpoint. SFTPGo uses by default [Swagger UI](https://github.com/swagger-api/swagger-ui){:target="_blank"}, you can use another renderer just by copying it to the defined OpenAPI path.

You can also explore the schema on [Stoplight](https://sftpgo.stoplight.io/docs/sftpgo/openapi.yaml){:target="_blank"}.

You can generate your own REST API client in your preferred programming language, or even bash scripts, using an OpenAPI generator such as [swagger-codegen](https://github.com/swagger-api/swagger-codegen){:target="_blank"} or [OpenAPI Generator](https://openapi-generator.tech/){:target="_blank"}.
