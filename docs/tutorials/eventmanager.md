# Event Manager

The Event Manager allows an administrator to configure HTTP notifications, commands execution, email notifications and carry out certain server operations based on server events or schedules. More details [here](../eventmanager.md).

Let's see some common use cases.

## Preliminary Note

We will use email actions in the following paragraphs, so let's assume you have a working SMTP configuration.
You can adapt the following snippet to configure an SMTP server using environment variables.

```shell
SFTPGO_SMTP__HOST="your smtp server host"
SFTPGO_SMTP__FROM="SFTPGo <sftpgo@example.com>"
SFTPGO_SMTP__USER=sftpgo@example.com
SFTPGO_SMTP__PASSWORD="your password"
SFTPGO_SMTP__AUTH_TYPE=1 # change based on what your server supports
SFTPGO_SMTP__ENCRYPTION=2 # change based on what your server supports
```

SFTPGo supports several placeholders for event actions. You can see all supported placeholders by clicking on the "info" icon at the top right of the add/update action page.

## Daily backups

You can schedule SFTPGo data backups (users, folders, groups, admins etc.) on a regular basis, such as daily.

From the WebAdmin expand the `Event Manager` section, select `Event actions` and add a new action.
Create an action named `backup` and set the type to `Backup`.

![Backup action](../assets/img/backup-action.png){data-gallery="backup"}

Create another action named `backup notification`, set the type to `Email` and fill the recipient/s.
As email subject set `Backup {{.StatusString}}`. The `{{.StatusString}}` placeholder will be expanded to `OK` or `KO`.
As email body set `Backup done {{.ErrorString}}`. The error string will be empty if no errors occur.

![Backup notification action](../assets/img/backup-notification-action.png){data-gallery="backup-notification"}

Now select `Event rules` and create a rule named `Daily backup`, select `Schedule` as trigger and schedule a backup at midnight UTC time.

![Daily backup schedule](../assets/img/daily-backup-schedule.png){data-gallery="schedule"}

As actions select `backup` and `backup notification`.

![Daily backup actions](../assets/img/daily-backup-actions.png){data-gallery="backup-actions"}

Done! SFTPGo will make a new backup every day and you will receive an email with the status of the backup. The backup will be saved on the server side in the configured backup directory. The backup files will have names like this `backup_<week day>_<hour>.json`.

## Automatically create a folder structure

Suppose you want to automatically create the folders `in` and `out` when you create new users.

From the WebAdmin expand the `Event Manager` section, select `Event actions` and add a new action.
Create an action named `create dirs`, with the settings you can see in the following screen.

![Create dirs action](../assets/img/create-dirs-action.png){data-gallery="create-dir-action"}

Create another action named `create dirs failure notification`, set the type to `Email` and fill the recipient/s.
As email subject set `Unable to create dirs for user {{.ObjectName}}`.
As email body set `Error: {{.ErrorString}}`.

![Create dirs notification](../assets/img/create-dirs-failure-notification.png){data-gallery="create-dir-failure-action"}

Now select `Event rules` and create a rule named `Create dirs for users`, select `Provider event` as trigger, `add` as provider event and `user` as object filters.

![Create dirs rule](../assets/img/create-dirs-rule.png){data-gallery="create-dir-rule"}

As actions select `create dirs` and `create dirs failure notification`, check `Is failure action` for the notification action.
This way you will only be notified by email if an error occurs.

![Create dirs rule actions](../assets/img/create-dirs-rule-actions.png){data-gallery="create-dir-rule-action"}

Done! Create a new user and check that the defined directories are automatically created.

## Upload notifications

Let's see how you can receive an email notification after each upload and, optionally, the uploaded file as well.

From the WebAdmin expand the `Event Manager` section, select `Event actions` and add a new action.
Create an action named `upload notification` with the settings you can see in the following screen.

![Upload notification action](../assets/img/upload-notification.png){data-gallery="upload-notification"}

You can optionally add the uploaded file as an attachment but note that SFTPGo allows you to attach a maximum of 10MB. Then the action will fail for files bigger than 10MB.

Now select `Event rules` and create a rule named `Upload rule`, select `Filesystem events` as trigger and `upload` as filesystem event.
You can also filters events based on protocol, user and group name, filepath shell-like patterns, file size. We omit these additional filters for simplicity.

![Upload rule](../assets/img/upload-rule.png){data-gallery="upload-rule"}

As actions, select `upload notification`.
Done! Try uploading a new file and you will receive the configured email notification.

## Recycle Bin

Let's see how we can configure a Recycle Bin style function where files are not deleted strait away, but instead moved to a separate folder.

To easily apply the Recycle Bin to multiple users we will create a virtual folder and a group, this way all users who belong to the group will have a Recycle Bin.

Create a virtual folder named `recycle` with the settings you can see in the following screen.

![Recycle folder](../assets/img/recycle-folder.png){data-gallery="recycle-folder"}

Create a group named `recycle` with the settings you can see in the following screen.

![Recycle group](../assets/img/recycle-group.png){data-gallery="recycle-group"}

Make your users members of the `recycle` group.

From the WebAdmin expand the `Event Manager` section, select `Event actions` and add a new action.
Create an action named `move to recycle` with the settings you can see in the following screen.

![Recycle move action](../assets/img/recycle-move-action.png){data-gallery="recycle-move-action"}

Now select `Event rules` and create a rule named `Recycle rule`, select `Filesystem events` as trigger, `pre-delete` as filesystem event and exclude the `/recycle` path.

![Recycle rule](../assets/img/recycle-rule.png){data-gallery="recycle-rule"}

![Recycle rule exclude path](../assets/img/recycle-rule-path.png){data-gallery="recycle-rule-path"}

As actions, select `move to recycle` and set `Execute sync`.

Done! Try deleting a file, it will be moved to the Recycle Bin.

You can also add a scheduled event rule to automatically delete files older than a configurable time from the `/recycle` folder.
