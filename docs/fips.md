---
description: "Run SFTPGo in FIPS 140-3 mode with a FIPS-validated cryptographic module. FIPS builds for Linux, Windows, and Docker use only approved algorithms."
---

# FIPS 140-3 mode

SFTPGo can run in **FIPS 140-3 mode**, where every security boundary uses only cryptography provided by a FIPS-validated module. This is intended for deployments that must satisfy a FIPS 140-3 requirement (US federal, defense contractors, regulated industries).

## What FIPS mode provides

FIPS builds are compiled against the **Go Cryptographic Module v1.0.0** (CMVP certificate [#5247](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/5247){:target="_blank"}, [CAVP A6650](https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program/details?product=19371){:target="_blank"}). The module is a self-contained Go module — not an OpenSSL wrapper. In a FIPS build, FIPS mode is **active by default**: the module runs its power-on self-tests at startup and routes the approved algorithms through the validated implementation.

:warning: **A license with the FIPS feature enabled is required.** FIPS mode is part of the [Ultimate tier](https://sftpgo.com/on-premises){:target="_blank"}: a FIPS build refuses to start unless the active license includes the FIPS feature. Make sure your license covers FIPS before deploying a FIPS build.

## Installing a FIPS build

FIPS builds are distributed alongside the standard builds. See [Installation](installation.md) for the package repositories and [Docker](docker.md) for the image variants.

- **Linux (APT/YUM):** install the `sftpgo-fips` package, plus `sftpgo-plugins-fips` for the bundled plugins.
- **Windows:** download the [FIPS installer](https://download.sftpgo.com/windows/sftpgo_windows_fips_x86_64.exe){:target="_blank"}.
- **Docker:** use the `-fips` image variants (see [Image variants](docker.md#image-variants)).

No runtime configuration is required: installing a FIPS build is what selects FIPS mode.

## Behavior in FIPS mode

In FIPS mode SFTPGo restricts itself to approved algorithms automatically; most of this needs no configuration.

### Password hashing

Password hashing uses **PBKDF2-SHA256**, which is FIPS-approved.

### Data-at-rest encryption (CryptFs)

[Encrypted folders](dare.md) use **AES-256-GCM** in FIPS mode. On non-FIPS builds without AES hardware acceleration, SFTPGo may instead use ChaCha20-Poly1305; files encrypted that way cannot be read under FIPS mode and must be re-encrypted first. On hardware with AES-NI — the common case — existing files are already AES-256-GCM and are unaffected.

### TLS (FTPS, WebDAV, HTTPS, REST)

Only approved TLS 1.2 and 1.3 cipher suites and signature algorithms are negotiated. Certificates may use **RSA** (2048-bit or larger), **ECDSA** (P-256/P-384/P-521), or **Ed25519** keys. Clients restricted to non-approved algorithms — notably ChaCha20-Poly1305, or TLS 1.1 and earlier — cannot connect.

### SSH (SFTP)

Only approved SSH algorithms are negotiated. Host keys and user public keys may use **RSA**, **ECDSA**, or **Ed25519**; legacy DSA keys and SHA-1-based algorithms are not accepted. Modern key exchanges and AES-GCM/AES-CTR ciphers remain available, so current SSH clients connect without changes; only clients pinned to non-approved algorithms (for example ChaCha20-Poly1305) need reconfiguration.
