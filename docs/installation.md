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

APT and YUM repositories are available through [Oregon State University's](https://osuosl.org/){:target="_blank"} free mirroring service. Special thanks to Lance Albertson, Director of the Oregon State University Open Source Lab.

SFTPGo is included in some distro repositories, we only document packages that we maintain directly.

### Ubuntu

For Ubuntu a PPA is [available](https://launchpad.net/~sftpgo/+archive/ubuntu/sftpgo){:target="_blank"}.

```shell
sudo add-apt-repository ppa:sftpgo/sftpgo
sudo apt update
sudo apt install sftpgo
```

After installation SFTPGo should already be running with default configuration and configured to start automatically at boot, check its status using the following command:

```shell
systemctl status sftpgo
```

### APT repo

Supported distributions:

- Debian 10 "buster"
- Debian 11 "bullseye"
- Debian 12 "bookworm"
- Debian 13 "trixie"

Import the public key used by the package management system:

```shell
curl -sS https://ftp.osuosl.org/pub/sftpgo/apt/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/sftpgo-archive-keyring.gpg
```

If you receive an error indicating that `gnupg` is not installed, you can install it using the following command:

```shell
sudo apt install gnupg
```

Create the SFTPGo source list file:

```shell
CODENAME=`lsb_release -c -s`
echo "deb [signed-by=/usr/share/keyrings/sftpgo-archive-keyring.gpg] https://ftp.osuosl.org/pub/sftpgo/apt ${CODENAME} main" | sudo tee /etc/apt/sources.list.d/sftpgo.list
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
curl -sS https://ftp.osuosl.org/pub/sftpgo/yum/${ARCH}/sftpgo.repo | sudo tee /etc/yum.repos.d/sftpgo.repo
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
sudo rpm --import https://ftp.osuosl.org/pub/sftpgo/apt/gpg.key
```

Add the SFTPGo repository:

```shell
ARCH=`uname -m`
sudo zypper addrepo -f "https://ftp.osuosl.org/pub/sftpgo/yum/${ARCH}" sftpgo
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
