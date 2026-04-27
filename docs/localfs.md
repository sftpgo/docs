---
description: "Configure the local filesystem storage backend in open-source SFTPGo with per-user home directories, cross-device rename support, and file path validation."
---

# Local filesystem

SFTPGo allow to restrict users to a specified directory on local filesystem, their "Home Dir".

To add a mapping for a directory outside the Home Dir you have to create a virtual folder, symbolic links outside the home directory are not allowed.
