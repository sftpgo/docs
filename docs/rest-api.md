# REST API

SFTPGo provides a comprehensive REST API that enables full programmatic control of the system. The API is divided into two distinct areas:

- Admin API: When authenticated as an administrator, you can manage users, groups, virtual folders, quotas, and system settings.
- User API: When authenticated as a regular user, you can perform file uploads and downloads, manage shares, and handle other file-related operations programmatically.

## Security and Authentication

The REST API is secured using JSON Web Tokens (JWT) or API key authentication. We strongly recommend to enable HTTPS to protect data in transit. For enhanced security, client certificate authentication can also be configured in conjunction with JWT, adding an additional layer of trust verification.

## Configuration

The REST API feature can be enabled or disabled through the `httpd` configuration section on a per-binding basis, giving you flexibility in controlling API availability.

## Documentation and Client Generation

Complete and up-to-date API documentation is available in [OpenAPI format](https://sftpgo.com/rest-api){:target="_blank"}, allowing developers to explore all endpoints, request parameters, and response formats.

Using this OpenAPI specification, you can easily generate client libraries or custom integrations in your preferred programming languages. Tools such as [Swagger Codegen](https://github.com/swagger-api/swagger-codegen){:target="_blank"} and [OpenAPI Generator](https://openapi-generator.tech/){:target="_blank"} support generating clients ranging from Python, Java, and JavaScript to simple bash scripts, streamlining the integration process.

## Examples

Below are practical examples using `curl` and `jq` to interact with the API.

### Admin API: Creating a User

To perform administrative tasks, you must first obtain an access token. Below are two examples: creating a user with direct configuration and creating a user that inherits settings from a group.

#### Authentication (Common Step)

First, obtain the JWT token using your admin credentials. You will need this token for all subsequent requests.

```shell
# SFTPGo endpoint and credentials
ENDPOINT="https://sftpgo.example.com"
ADMIN_USER="admin"
ADMIN_PASSWORD="your_admin_password"

# Get the JWT Token
TOKEN=$(curl --anyauth -s -u "${ADMIN_USER}:${ADMIN_PASSWORD}" \
  "${ENDPOINT}/api/v2/token" | jq -r .access_token)

echo "Token acquired."
```

#### Case A: Creating a Standalone User (No Group)

In this scenario, you explicitly define the file system settings (S3 bucket, region, and key prefix) directly in the user payload.

```shell
# Define the payload for a standalone user
# We configure the S3 backend explicitly here.
USER_PAYLOAD=$(cat <<EOF
{
  "status": 1,
  "username": "testuser_standalone",
  "password": "clear_text_complex_password",
  "permissions": {
    "/": ["*"]
  },
  "filesystem": {
    "provider": 1,
    "s3config": {
      "bucket": "testbucket",
      "region": "eu-central-1",
      "access_key": "myaccesskey",
      "access_secret": {
        "status": "Plain",
        "payload": "myaccesssecret"
      },
      "key_prefix": "users/testuser_standalone/"
    }
  }
}
EOF
)

# Create the user
curl -X POST \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "${USER_PAYLOAD}" \
  "${ENDPOINT}/api/v2/users"
```

#### Case B: Creating a User with Group Membership

In this scenario, the user inherits the file system configuration from a Primary Group. Note: The group specified in the payload (e.g., `BasicUsers`) must already exist in SFTPGo.

```shell
GROUP_NAME="BasicUsers"

# Define the payload for a group-based user
# Note:
# 1. We assign the user to a Primary Group (type: 1).
# 2. Since the Group handles the filesystem, we can pass dummy/invalid values 
#    for the user's explicit filesystem configuration.
USER_PAYLOAD=$(cat <<EOF
{
  "status": 1,
  "username": "testuser_group",
  "password": "clear_text_complex_password",
  "groups": [
    {
      "type": 1, 
      "name": "${GROUP_NAME}"
    }
  ],
  "permissions": {
    "/": ["*"]
  },
  "filesystem": {
    "provider": 1,
    "s3config": {
      "bucket": "invalid",
      "region": "invalid"
    }
  }
}
EOF
)

# Create the user
curl -X POST \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "${USER_PAYLOAD}" \
  "${ENDPOINT}/api/v2/users"
```

### User API: File Operations

Regular users can also use the API to manage their own files. The authentication endpoint for users is different from the admin one (`/api/v2/user/token`).

**Important**: URL Encoding When specifying file paths in the path query parameter (e.g., for files inside subdirectories or filenames with spaces), you must URL encode the value.

- Example: `my_dir/file.txt` becomes `my_dir%2Ffile.txt`
- Example: `my file.txt` becomes `my%20file.txt`

The following example shows how to list directories, upload a file, and download it back.

```shell
#!/bin/bash

# SFTPGo endpoint and user credentials
ENDPOINT="https://sftpgo.example.com"
USERNAME="testuser"
PASSWORD="clear_text_complex_password"

# 1. Get the User JWT Token
# Note the endpoint is /api/v2/user/token
TOKEN=$(curl --anyauth -s -u "${USERNAME}:${PASSWORD}" \
  "${ENDPOINT}/api/v2/user/token" | jq -r .access_token)

echo "User Token acquired."

# 2. List directory contents
echo "--- Home Directory Contents ---"
curl -s -H "Authorization: Bearer ${TOKEN}" "${ENDPOINT}/api/v2/user/dirs" | jq .
echo "-------------------------------"

# 3. Upload a file to a subdirectory
# We want to upload to: "my_subdir/uploaded_file.txt"
# We must URL encode the path separator '/' to '%2F'
# The path parameter becomes: "my_subdir%2Fuploaded_file.txt"

echo "Uploading file..."
echo "Hello SFTPGo" > local_file.txt

# 'mkdir_parents=true' ensures 'my_subdir' is created automatically if it doesn't exist
curl -X POST \
  -H "Authorization: Bearer ${TOKEN}" \
  -d "@local_file.txt" \
  "${ENDPOINT}/api/v2/user/files/upload?path=my_subdir%2Fuploaded_file.txt&mkdir_parents=true"

# 4. Download the file from the subdirectory
# Again, we use the URL encoded path: "my_subdir%2Fuploaded_file.txt"

echo "Downloading file..."
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${ENDPOINT}/api/v2/user/files?path=my_subdir%2Fuploaded_file.txt" > downloaded_file.txt

echo "Download complete. Content:"
cat downloaded_file.txt
```
