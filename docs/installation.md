---
description: "Install open-source SFTPGo on Linux, Windows, macOS, FreeBSD, or Docker. Available via APT/YUM repos, official binaries, and Docker images."
---

# Installation

SFTPGo runs on small embedded devices or large Kubernetes clusters. On Linux, Windows, macOS, FreeBSD. Other *BSD variants should work too.

If you'd prefer to focus on your core business without worrying about the maintenance and security of your file transfer solution, consider opting for our fully managed [SaaS offerings](https://sftpgo.com/saas){:target="_blank"}. With a dedicated installation tailored specifically to your needs, you'll receive a secure, high-performance solution fully managed by us, the authors of SFTPGo. We handle everything from security patches to upgrades, ensuring your service runs smoothly at all times.

## Requirements

The only (optional) requirement is a suitable SQL server to use as data provider:

- upstream supported versions of PostgreSQL, MySQL and MariaDB.
- CockroachDB stable.

You can remove this requirement by using an embedded SQLite, bolt or in memory data provider.

## AWS

SFTPGo is available on [AWS Marketplace](https://aws.amazon.com/marketplace/seller-profile?id=6e849ab8-70a6-47de-9a43-13c3fa849335){:target="_blank"}.

Marketplace offerings are pre-configured with a specific data-provider but all of them can be reconfigured to use a different data-provider.

## Azure

SFTPGo is available on Azure Marketplace:

- [SFTPGo for Linux](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/eliamarzia1667381463185.sftpgo_linux){:target="_blank"}
- [SFTPGo for Windows](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/eliamarzia1667381463185.sftpgo_windows){:target="_blank"}
- [SFTPGo for AKS](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/eliamarzia1667381463185.sftpgo_aks){:target="_blank"}

## Google Cloud

SFTPGo is available on [Google Cloud Marketplace](https://console.cloud.google.com/marketplace/browse?filter=partner:SFTPGo%20Authors){:target="_blank"}.

## Linux

SFTPGo is included in some distro repositories, we only document packages that we maintain directly.

### APT repo

Supported distributions:

- Debian 10 "buster"
- Debian 11 "bullseye"
- Debian 12 "bookworm"
- Debian 13 "trixie"
- Ubuntu 20.04 "focal"
- Ubuntu 22.04 "jammy"
- Ubuntu 24.04 "noble"
- Ubuntu 26.04 "resolute"

Import the public key used by the package management system:

```shell
curl -sS https://oss.sftpgo.com/apt/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/sftpgo-archive-keyring.gpg
```

If you receive an error indicating that `gnupg` is not installed, you can install it using the following command:

```shell
sudo apt install gnupg
```

Create the SFTPGo source list file:

```shell
CODENAME=`lsb_release -c -s`
echo "deb [signed-by=/usr/share/keyrings/sftpgo-archive-keyring.gpg] https://oss.sftpgo.com/apt ${CODENAME} main" | sudo tee /etc/apt/sources.list.d/sftpgo.list
```

Reload the package database and install SFTPGo:

```shell
sudo apt update
sudo apt install sftpgo
```

### Yum repo

The YUM repository can be used on generic Red Hat based distributions as well as on Suse/OpenSuse.

#### Red Hat based distributions

Create the SFTPGo repository:

```shell
ARCH=`uname -m`
curl -sS https://oss.sftpgo.com/yum/${ARCH}/sftpgo.repo | sudo tee /etc/yum.repos.d/sftpgo.repo
```

Reload the package database and install SFTPGo:

```shell
sudo yum update
sudo yum install sftpgo
```

Start the SFTPGo service and enable it to start at system boot:

```shell
sudo systemctl start sftpgo
sudo systemctl enable sftpgo
```

#### Suse/OpenSUSE

Import the public key used by the package management system:

```shell
sudo rpm --import https://oss.sftpgo.com/yum/gpg.key
```

Add the SFTPGo repository:

```shell
ARCH=`uname -m`
sudo zypper addrepo -f "https://oss.sftpgo.com/yum/${ARCH}" sftpgo
```

Reload the package database and install SFTPGo:

```shell
sudo zypper refresh
sudo zypper install sftpgo
```

Start the SFTPGo service and enable it to start at system boot:

```shell
sudo systemctl start sftpgo
sudo systemctl enable sftpgo
```

### Arch Linux

SFTPGo is available via AUR:

- [sftpgo](https://aur.archlinux.org/packages/sftpgo/){:target="_blank"}. This package follows stable releases. It requires `git`, `gcc` and `go` to build.
- [sftpgo-bin](https://aur.archlinux.org/packages/sftpgo-bin/){:target="_blank"}. This package follows stable releases downloading the prebuilt linux binary from GitHub. It does not require `git`, `gcc` and `go` to build.
- [sftpgo-git](https://aur.archlinux.org/packages/sftpgo-git/){:target="_blank"}. This package builds and installs the latest git `main` branch. It requires `git`, `gcc` and `go` to build.

## Windows

You can download and install the Windows installer from our [release](https://github.com/drakkan/sftpgo/releases){:target="_blank"} page. The installer will register and run SFTPGo as a Windows service.

Other options:

- The portable [release](https://github.com/drakkan/sftpgo/releases){:target="_blank"} to run SFTPGo on demand.
- The [winget](https://docs.microsoft.com/en-us/windows/package-manager/winget/install){:target="_blank"} package to install and run SFTPGo as a Windows service: `winget install -e --id drakkan.SFTPGo`.
- The [Chocolatey package](https://community.chocolatey.org/packages/sftpgo){:target="_blank"} to install and run SFTPGo as a Windows service.

## macOS

SFTPGo is available as Homebrew [Formula](https://formulae.brew.sh/formula/sftpgo){:target="_blank"}.

## FreeBSD

SFTPGo is included in FreeBSD [Ports](https://www.freshports.org/ftp/sftpgo){:target="_blank"}.

## Docker

SFTPGo provides an official Docker image, more [details](docker.md).

## Service account

Run the service with the least privileges it needs: access to the configured home directories and to its own data, and nothing else.

The Linux packages we maintain register a systemd service running under the dedicated, unprivileged `sftpgo` account, which owns the packaged data directories, and the Docker images run as user and group `1000`. On Windows the service is registered under `LocalSystem`: a dedicated account is set through the Windows service configuration and then needs access to the SFTPGo data directory and to the home directories. The installer removes and registers the service again on upgrade, restoring `LocalSystem`, so set the account again after each update.

Running SFTPGo under a privileged account is supported and a few deployments need it. On Unix-like systems, setting per-user ownership on uploaded files is one such case. Every file operation is carried out with the privileges of the service account, so provision the home directories and the paths leading to them as described in [Local filesystem](localfs.md).

:warning: SFTPGo has no operating system identity per session: it is a single process and does not switch to a Unix user per connection, so the permissions granted to the virtual user are the boundary between clients rather than the ones the operating system enforces. A privileged account widens what the service can reach and leaves that boundary as the only one: hooks, command actions and plugins run with the privileges of the service on every platform.

On Unix-like systems a client granted `chmod` and `chown`, both part of the default `*` permission set, can set any mode, `setuid` included, and any owner on the files in its own tree, which on a host where those clients can also execute files amounts to granting them the privileges of the service account. Under an unprivileged account the chown fails and a setuid bit confers no more than that account. On Windows the mode maps to the read-only attribute and ownership is left untouched, so the per-user ownership mapping applies to Unix-like systems only.

Where a privileged account is required:

- `setstat_mode` set to `1` makes SFTPGo ignore attribute change requests
- removing `chmod` and `chown` from the users' permissions rejects them
- on Unix-like systems, mounting the filesystem that holds the home directories with `nosuid` makes setuid binaries ineffective whatever their origin

On Unix-like systems SFTPGo logs a warning at startup when it runs with an effective uid of `0`.
