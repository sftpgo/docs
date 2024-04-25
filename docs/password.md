# Supported Password Hashing Algorithms

SFTPGo can verify passwords in several formats and uses, by default, the `bcrypt` algorithm to hash passwords in plain-text before storing them inside the data provider. Each hashing algorithm is identified by a prefix.
Supported hash algorithms:

- bcrypt, prefix `$2a$`
- argon2id, prefix `$argon2id$`
- PBKDF2 sha1, prefix `$pbkdf2-sha1$`
- PBKDF2 sha256, prefix `$pbkdf2-sha256$`
- PBKDF2 sha512, prefix `$pbkdf2-sha512$`
- PBKDF2 sha256 with base64 salt, prefix `$pbkdf2-b64salt-sha256$`
- MD5 crypt, prefix `$1$`
- MD5 crypt APR1, prefix `$apr1$`
- SHA256 crypt, prefix `$5$`
- SHA512 crypt, prefix `$6$`
- MD5 digest, prefix `{MD5}`
- SHA256 digest, prefix `{SHA256}`
- SHA512 digest, prefix `{SHA512}`

If you set a password with one of these prefixes it will not be hashed.
When users log in, if their passwords are stored with anything other than the preferred algorithm, SFTPGo will automatically upgrade the algorithm to the preferred one.
