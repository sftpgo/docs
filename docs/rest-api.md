# REST API

SFTPGo provides a comprehensive REST API that enables full programmatic control over both administrative tasks and user operations. This API allows administrators to manage users, groups, virtual folders, quotas, and system settings, while end users can perform file uploads, downloads, share management, and more — all remotely and securely.

## Security and Authentication

The REST API is secured using JSON Web Tokens (JWT) or API key authentication. We strongly recommend to enable HTTPS to protect data in transit. For enhanced security, client certificate authentication can also be configured in conjunction with JWT, adding an additional layer of trust verification.

## Configuration

The REST API feature can be enabled or disabled through the `httpd` configuration section on a per-binding basis, giving you flexibility in controlling API availability.

## Documentation and Client Generation

Complete and up-to-date API documentation is available in [OpenAPI format](https://sftpgo.com/rest-api){:target="_blank"}. This allows developers to explore the full range of endpoints, request parameters, and response formats.

Using this OpenAPI specification, you can easily generate client libraries or custom integrations in your preferred programming languages. Tools such as [Swagger Codegen](https://github.com/swagger-api/swagger-codegen){:target="_blank"} and [OpenAPI Generator](https://openapi-generator.tech/){:target="_blank"} support generating clients ranging from Python, Java, and JavaScript to simple bash scripts, streamlining the integration process.
