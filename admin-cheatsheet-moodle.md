# Admin Cheatsheet For Moodle Production

Frequently used production administration commands for Moodle installs and cutovers.

## Production CLI Block

Adjust the path for the production code location. Moodle 5.1 still uses root `admin/cli`; the web cron endpoint is under `public/admin/cron.php`.

```bash
cd /var/www/moodle

sudo -u www-data php admin/cli/maintenance.php --enable
sudo -u www-data php admin/cli/upgrade.php --non-interactive
sudo -u www-data php admin/cli/purge_caches.php
sudo -u www-data php admin/cli/check_database_schema.php
sudo -u www-data php admin/cli/maintenance.php --disable
sudo -u www-data php admin/cli/cron.php --keep-alive=0
sudo -u www-data php admin/cli/checks.php
```

For Moodle 3.11, run cron without `--keep-alive=0`:

```bash
sudo -u www-data php admin/cli/cron.php
```

## Production Mail Smoke Test

After production cutover or SMTP changes, test both Moodle's core outgoing mail page and the eMail Test plugin:

```text
https://<production-moodle-host>/admin/testoutgoingmailconf.php
https://<production-moodle-host>/local/mailtest/
```

If the test pages report success but messages take a few minutes to arrive, check cron and queued ad hoc task processing:

```bash
cd /var/www/moodle

sudo -u www-data php admin/cli/cron.php --keep-alive=0
sudo -u www-data php admin/cli/adhoc_task.php --execute --keep-alive=59
sudo -u www-data php admin/cli/adhoc_task.php --failed --showdebugging
```

The test pages can confirm Moodle hands the message to the mail server. Final inbox delivery may still be delayed by Moodle ad hoc processing, SMTP relay handling, or recipient-side filtering.

## Optional Trash Cleanup

Do not include this in routine health checks. It permanently empties Moodle's file trash and may remove files users still expect to recover.

Production form:

```bash
cd /var/www/moodle
sudo -u www-data php admin/cli/scheduled_task.php --execute='\core\task\file_trash_cleanup_task'
```
