# Runbook: Export Old Production Moodle To Local Docker

Prepared: 2026-08-29

Source site: `gen3moodle.bilkent.edu.tr`

Source SSH user: `yan`

Purpose:

1. Export the current production Moodle 3.6.10 site.
2. Download the database, Moodle code evidence, config, and Moodle data to the local upgrade workspace.
3. Prepare the local files needed for the Docker upgrade rehearsal.

This runbook stops at the local Docker handoff. It does not cover deployment to the new Debian 12 production server.

Before starting, complete:

```text
pre-upgrade-production-preparation-runbook.md
```

That preparation runbook covers old theme cleanup, stale plugin removal, RecordRTC checks, and the cleaned additional-plugin baseline.

## 1. Measure Source Data Size

Run these from your local machine:

```bash
ssh yan@gen3moodle.bilkent.edu.tr
```

Once connected:

```bash
sudo -v
date
hostname
hostnamectl
df -hT
```

Find Moodle's `config.php`. The known path is `/var/www/html/config.php`, but confirm it:

```bash
sudo find /var/www -maxdepth 4 -name config.php -print
ls -la /var/www/html
```

Read the relevant lines:

```bash
sudo grep -n "wwwroot\|dataroot\|dbtype\|dbhost\|dbname\|dbuser\|prefix" /var/www/html/config.php
```

Set these manually from the `config.php` output:

```bash
MOODLE_CODE=/var/www/html
MOODLE_DATA=/PATH/FROM/CFG_DATAROOT
MOODLE_DB=DATABASE_NAME_FROM_CFG_DBNAME

echo "MOODLE_CODE=$MOODLE_CODE"
echo "MOODLE_DATA=$MOODLE_DATA"
echo "MOODLE_DB=$MOODLE_DB"
```

Measure Moodle code and Moodle data:

```bash
sudo du -sh "$MOODLE_CODE"
sudo du -sh "$MOODLE_DATA"
sudo du -sh "$MOODLE_DATA"/filedir "$MOODLE_DATA"/cache "$MOODLE_DATA"/localcache "$MOODLE_DATA"/temp "$MOODLE_DATA"/trashdir 2>/dev/null
```

Show the largest first-level directories:

```bash
sudo du -xh --max-depth=1 "$MOODLE_DATA" | sort -h
sudo du -xh --max-depth=1 "$MOODLE_CODE" | sort -h
```

Measure the database logical size:

```bash
sudo mysql -NBe "SELECT table_schema, ROUND(SUM(data_length + index_length)/1024/1024, 1) AS size_mb FROM information_schema.tables WHERE table_schema = '${MOODLE_DB}' GROUP BY table_schema;"
```

Optional: list the largest Moodle database tables:

```bash
sudo mysql -NBe "SELECT table_name, ROUND((data_length + index_length)/1024/1024, 1) AS size_mb, table_rows FROM information_schema.tables WHERE table_schema = '${MOODLE_DB}' ORDER BY data_length + index_length DESC LIMIT 30;"
```

Check MariaDB version:

```bash
sudo mysql -e "SELECT VERSION();"
```

Optional: estimate compressed database dump size:

```bash
sudo mysqldump --single-transaction --quick --routines --triggers --events "$MOODLE_DB" | gzip -1 | wc -c
```

Check available space for staging exports:

```bash
df -h /tmp /var /root "$MOODLE_DATA"
```

If there is not enough free disk space for a Moodle data archive on the source server, transfer Moodle data directly with FileZilla/FTP/SFTP or `rsync`, excluding disposable cache directories.

## 2. Run Source Moodle Health Checks

Run these checks before putting the source site into maintenance mode for the final export.

Start with a normal Moodle cron run while the site is still available:

```bash
cd "$MOODLE_CODE"
sudo -u www-data php admin/cli/cron.php
```

In Moodle 3.6, `admin/cli/checks.php` may not exist. If it exists after a later local upgrade stage, use it; on the old production source, rely on the checks below.

Check database schema:

```bash
cd "$MOODLE_CODE"
sudo -u www-data php admin/cli/check_database_schema.php
```

List storage engines:

```bash
cd "$MOODLE_CODE"
sudo -u www-data php admin/cli/mysql_engine.php --list
```

If any tables are `MyISAM`, plan an InnoDB conversion before continuing:

```bash
cd "$MOODLE_CODE"
sudo -u www-data php admin/cli/mysql_engine.php --engine=InnoDB
```

Do not run the engine conversion blindly. Confirm the `--list` output first.

Check current database collation state:

```bash
sudo mysql "$MOODLE_DB" -e "
SELECT table_collation, COUNT(*) AS tables
FROM information_schema.tables
WHERE table_schema = '$MOODLE_DB'
GROUP BY table_collation
ORDER BY tables DESC, table_collation;
"
```

If production is not already using `utf8mb4_unicode_ci`, note it for the local upgrade rehearsal. The local Moodle 3.11 rehearsal converted successfully on 2026-08-29 with:

```bash
sudo -u www-data php admin/cli/mysql_collation.php --collation=utf8mb4_unicode_ci
```

Do not run the collation conversion on live production unless you have a fresh backup and a maintenance window.

List compressed InnoDB rows:

```bash
cd "$MOODLE_CODE"
sudo -u www-data php admin/cli/mysql_compressed_rows.php --list
```

Check core version, theme, and maintenance flags:

```bash
sudo mysql "$MOODLE_DB" -e "
SELECT name, value
FROM ${PREFIX}config
WHERE name IN ('release','version','theme','themelist','maintenance_enabled');
"
```

Check remaining contributed activity module instance counts:

```bash
sudo mysql "$MOODLE_DB" -e "
SELECT m.name, COUNT(cm.id) AS instances
FROM ${PREFIX}modules m
LEFT JOIN ${PREFIX}course_modules cm ON cm.module = m.id
WHERE m.name IN ('checklist','hvp','pdfannotator','questionnaire','scheduler','turnitintooltwo')
GROUP BY m.name
ORDER BY m.name;
"
```

Check kept contributed plugin versions:

```bash
sudo mysql "$MOODLE_DB" -e "
SELECT plugin, value AS version
FROM ${PREFIX}config_plugins
WHERE name = 'version'
  AND plugin IN (
    'mod_checklist',
    'mod_hvp',
    'mod_pdfannotator',
    'mod_questionnaire',
    'mod_scheduler',
    'mod_turnitintooltwo',
    'plagiarism_turnitin',
    'local_mailtest'
  )
ORDER BY plugin;
"
```

Check old core theme leftovers:

```bash
sudo mysql "$MOODLE_DB" -e "
SELECT plugin, name, value
FROM ${PREFIX}config_plugins
WHERE plugin IN ('theme_bootstrapbase','theme_clean','theme_more');
"
```

If old theme config rows remain after the themes have been uninstalled and theme assignments have been cleared, remove them before export:

```bash
sudo mysql "$MOODLE_DB" -e "
DELETE FROM ${PREFIX}config_plugins
WHERE plugin IN ('theme_bootstrapbase','theme_clean','theme_more');
"
```

Then purge caches:

```bash
cd "$MOODLE_CODE"
sudo -u www-data php admin/cli/purge_caches.php
```

Check scheduled task failures:

```bash
sudo mysql "$MOODLE_DB" -e "
SELECT component, classname, faildelay, nextruntime
FROM ${PREFIX}task_scheduled
WHERE faildelay > 0
ORDER BY faildelay DESC;
"
```

Check pending ad hoc tasks:

```bash
sudo mysql "$MOODLE_DB" -e "
SELECT component, classname, COUNT(*) AS adhoc_tasks
FROM ${PREFIX}task_adhoc
GROUP BY component, classname
ORDER BY adhoc_tasks DESC;
"
```

If ad hoc tasks are pending, run cron again while maintenance mode is still disabled:

```bash
cd "$MOODLE_CODE"
sudo -u www-data php admin/cli/cron.php
```

For the 2026-08-29 local Moodle 3.11 rehearsal, the following post-upgrade checks were good:

```text
Database structure is ok.
Collation conversion to utf8mb4_unicode_ci completed with errors: 0.
Old theme config rows for bootstrapbase/clean/more were empty after cleanup.
No scheduled tasks had faildelay > 0.
Remaining activity counts:
  checklist          22
  hvp                4
  pdfannotator       124
  questionnaire      174
  scheduler          117
  turnitintooltwo    60
```

## 3. Create Local Directory Layout

On your local machine:

```bash
mkdir -p "$HOME/Moodle/gen3moodle-upgrade-2026"/{source-export,docker-work,stage-exports,production-import}
cd "$HOME/Moodle/gen3moodle-upgrade-2026"
```

Recommended layout:

```text
gen3moodle-upgrade-2026/
  source-export/
    moodle-db-source.sql.gz
    moodle-code-source.tgz
    moodledata-source/
    config.php.source
    source-inventory.txt
  docker-work/
  stage-exports/
    3.11.18-after/
    after-4.3.x/
    after-5.1.x/
  production-import/
```

## 4. Capture Source Inventory

On the source server:

```bash
ssh yan@gen3moodle.bilkent.edu.tr
```

Set or confirm the export directory:

```bash
EXPORT_DIR=/home/yan/gen3moodle-export
sudo mkdir -p "$EXPORT_DIR"
sudo chown -R yan:yan "$EXPORT_DIR"
chmod 700 "$EXPORT_DIR"
```

Capture system, web, PHP, MariaDB, and Moodle path/version evidence:

```bash
{
  echo "Captured at: $(date)"
  echo
  echo "== Host =="
  hostname
  hostnamectl
  echo
  echo "== Disk =="
  df -hT
  echo
  echo "== Apache =="
  apache2 -v 2>/dev/null || apache2ctl -v 2>/dev/null || true
  echo
  echo "== PHP =="
  php -v
  php -m
  echo
  echo "== MariaDB/MySQL Client =="
  mysql --version
  echo
  echo "== MariaDB Server =="
  sudo mysql -e "SELECT VERSION();"
  echo
  echo "== Moodle Config Paths =="
  sudo grep -n "wwwroot\|dataroot\|dbtype\|dbhost\|dbname\|dbuser\|prefix" /var/www/html/config.php
  echo
  echo "== Moodle Version =="
  sudo -u www-data php /var/www/html/admin/cli/cfg.php --name=release 2>/dev/null || true
  sudo -u www-data php /var/www/html/admin/cli/cfg.php --name=version 2>/dev/null || true
} > "$EXPORT_DIR/source-inventory.txt"
```

Check it:

```bash
less "$EXPORT_DIR/source-inventory.txt"
ls -lh "$EXPORT_DIR/source-inventory.txt"
```

Capture Moodle config into the same export directory:

```bash
sudo cp /var/www/html/config.php "$EXPORT_DIR/config.php.source"
sudo chown yan:yan "$EXPORT_DIR/config.php.source" "$EXPORT_DIR/source-inventory.txt"
chmod 600 "$EXPORT_DIR/config.php.source"
```

## 5. Put Source Site In Maintenance Mode And Stop Cron

On the source server:

```bash
ssh yan@gen3moodle.bilkent.edu.tr
cd /var/www/html
sudo -u www-data php admin/cli/maintenance.php --enable
```

Discover Moodle cron:

```bash
sudo crontab -l
sudo crontab -u www-data -l
systemctl list-timers | grep -i moodle
```

On this server, the `www-data` crontab showed:

```cron
*/5 * * * * /usr/bin/php /var/www/html/admin/cli/cron.php > /dev/null
```

Stop cron during the export if no other cron jobs must continue:

```bash
sudo systemctl stop cron
sudo systemctl status cron
ps aux | grep '[a]dmin/cli/cron.php'
```

Expected:

- `cron.service` shows `Active: inactive (dead)`.
- The process check returns no Moodle cron process.

If system cron must remain running, edit only the `www-data` crontab:

```bash
sudo crontab -u www-data -e
```

Comment out:

```cron
# */5 * * * * /usr/bin/php /var/www/html/admin/cli/cron.php > /dev/null
```

Recommended export order:

```text
1. Database dump
2. Moodledata copy
3. Code/config copy
```

## 6. Export Database And Moodle Code On Source Server

Set values from `config.php`:

```bash
MOODLE_CODE=/var/www/html
MOODLE_DATA=/PATH/FROM/CFG_DATAROOT
MOODLE_DB=DATABASE_NAME_FROM_CFG_DBNAME
EXPORT_DIR=/home/yan/gen3moodle-export

mkdir -p "$EXPORT_DIR"
df -h "$EXPORT_DIR"
```

Create the database dump:

```bash
sudo mysqldump --single-transaction --quick --routines --triggers --events "$MOODLE_DB" | gzip -1 > "$EXPORT_DIR/moodle-db-source.sql.gz"
```

If MariaDB requires an interactive root password:

```bash
sudo mysqldump -u root -p --single-transaction --quick --routines --triggers --events "$MOODLE_DB" | gzip -1 > "$EXPORT_DIR/moodle-db-source.sql.gz"
```

Create the Moodle code archive as evidence and rollback material:

```bash
sudo tar -C /var/www -czf "$EXPORT_DIR/moodle-code-source.tgz" html
```

Copy config:

```bash
sudo cp "$MOODLE_CODE/config.php" "$EXPORT_DIR/config.php.source"
```

Make export files downloadable by `yan`:

```bash
sudo chown -R yan:yan "$EXPORT_DIR"
chmod 600 "$EXPORT_DIR/config.php.source"
ls -lh "$EXPORT_DIR"
```

Verify:

```bash
gzip -t "$EXPORT_DIR/moodle-db-source.sql.gz"
tar -tzf "$EXPORT_DIR/moodle-code-source.tgz" | head
```

## 7. Export Moodle Data

Do not include disposable cache directories in the local upgrade copy.

If the source server has enough free space, create an archive:

```bash
sudo tar \
  --exclude='./cache' \
  --exclude='./localcache' \
  --exclude='./temp' \
  --exclude='./trashdir' \
  -C "$MOODLE_DATA" \
  -czf "$EXPORT_DIR/moodledata-source.tgz" .

sudo chown yan:yan "$EXPORT_DIR/moodledata-source.tgz"
tar -tzf "$EXPORT_DIR/moodledata-source.tgz" | head
```

If the source server does not have enough free space, download Moodle data directly to:

```text
$HOME/Moodle/gen3moodle-upgrade-2026/source-export/moodledata-source/
```

Exclude or skip:

```text
cache/
localcache/
temp/
trashdir/
```

Do not skip:

```text
filedir/
```

After transfer, optionally create a local tarball on your machine:

```bash
cd "$HOME/Moodle/gen3moodle-upgrade-2026/source-export"
tar -cf moodledata-source.tar moodledata-source
tar -tf moodledata-source.tar | head
ls -lh moodledata-source.tar
```

## 8. Download Server-side Export Files

From your local machine:

```bash
cd "$HOME/Moodle/gen3moodle-upgrade-2026/source-export"
scp 'yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/moodle-db-source.sql.gz' .
scp 'yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/moodle-code-source.tgz' .
scp 'yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/config.php.source' .
scp 'yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/source-inventory.txt' .
```

If you created `moodledata-source.tgz` on the server:

```bash
scp 'yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/moodledata-source.tgz' .
```

For interrupted downloads, use `rsync`:

```bash
rsync -avP yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/moodle-db-source.sql.gz .
rsync -avP yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/moodle-code-source.tgz .
rsync -avP yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/config.php.source .
rsync -avP yan@gen3moodle.bilkent.edu.tr:/home/yan/gen3moodle-export/source-inventory.txt .
```

Verify local copies:

```bash
ls -lh
gzip -t moodle-db-source.sql.gz
tar -tzf moodle-code-source.tgz | head
chmod 600 config.php.source
less source-inventory.txt
```

If you downloaded Moodle data as files:

```bash
du -sh moodledata-source
find moodledata-source -type f | wc -l
```

## 9. Optional: Remove Server-side Export Files

Only remove server-side export files after the local copies have been downloaded and verified.

On the source server:

```bash
rm -i /home/yan/gen3moodle-export/moodle-db-source.sql.gz
rm -i /home/yan/gen3moodle-export/moodle-code-source.tgz
rm -i /home/yan/gen3moodle-export/moodledata-source.tgz
rm -i /home/yan/gen3moodle-export/config.php.source
rm -i /home/yan/gen3moodle-export/source-inventory.txt
rmdir /home/yan/gen3moodle-export
```

Skip `moodledata-source.tgz` if it was never created.

## 10. Re-enable Source Site If Needed

If the live source site should continue running while you rehearse locally:

```bash
ssh yan@gen3moodle.bilkent.edu.tr
cd /var/www/html
sudo -u www-data php admin/cli/maintenance.php --disable
sudo systemctl start cron
sudo systemctl status cron
```

If the source site is staying frozen for a final cutover export, leave maintenance mode enabled and cron stopped.

## 11. Handoff To Local Docker Upgrade

The local Docker workspace should use:

```text
source-export/moodle-db-source.sql.gz
source-export/moodle-code-source.tgz
source-export/moodledata-source/
source-export/config.php.source
```

Continue with the stage upgrade runbooks, beginning with:

```text
stage-1-runbook-moodle-3.6.10-to-3.11.18.md
```

Recommended local stage ladder:

```text
Stage A:
  Moodle 3.6.10 source restore
  PHP 7.3
  MariaDB 10.2.x
  Upgrade to Moodle 3.11.18

Stage B:
  Moodle 3.11.18
  PHP 8.0 or 8.1
  MariaDB 10.6 or 10.11
  Upgrade to Moodle 4.3.x

Stage C:
  Moodle 4.3.x
  PHP 8.2
  MariaDB 10.11
  Upgrade to Moodle 5.1.x
```

At the end of each stage, export a stage checkpoint.

For the final production import, keep:

```text
stage-exports/after-5.1.x/moodle-db.sql.gz
stage-exports/after-5.1.x/moodle-code.tgz
source-export/moodledata-source/ transferred with filtered rsync
```

## Data Size Summary Template

Fill this in after running the sizing commands:

```text
Source host:
Source OS:
Source Moodle code path:
Source Moodle data path:
Source database name:

Moodle code size:
Moodle data size:
Moodle filedir size:
Moodle DB logical size:
Compressed DB dump estimate:
Compressed moodledata estimate:

Available source disk space:
Available local disk space:
```
