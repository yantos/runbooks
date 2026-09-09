# Runbook: Deploy Upgraded Local Docker Moodle To Debian 13 Trixie Production

Prepared: 2026-08-29
Updated: 2026-09-02

Target: `<production-moodle-host>`, new production VM running Debian 13 Trixie

Purpose:

1. Transfer the upgraded Moodle database and Moodle data from the local Docker upgrade workspace.
2. Transfer the final upgraded artifacts to the new Debian 13 Trixie production server.
3. Install Git-managed Moodle 5.1.6 code, deploy the tested plugin/custom code, and import the upgraded site.

This runbook assumes the local Docker upgrade has already reached Moodle 5.1.6 on PHP 8.4 and MariaDB 11.8.

The final database export and production `config.php` template are already ready locally at:

```text
<active-upgrade-root>/production-import/moodle-db.sql.gz
<active-upgrade-root>/production-import/config.php
```

Safety rules:

- Do not import SQL into production until Moodle core and all required plugin code are in place.
- Do not run Moodle CLI upgrade/checks against production with missing plugin directories.
- Do not deploy the whole old Moodle code tree.
- Do not deploy from `source-export` or mutate anything under `source-export`.
- Do not edit anything under `additional-plugins`; copy from it into production staging only.
- Do not make `/var/www/moodle` writable by the web server after deployment.
- Do not run Composer as `sudo`; temporarily make the code tree writable by the deploy user, run Composer as that user, then lock permissions back down.
- Do not delete `/var/moodledata/filedir` or `/var/moodledata/trashdir`.
- Do not enable production cron until CLI and browser checks pass.

## 1. Confirm Final Local Docker State

On your local machine:

```bash
cd <active-upgrade-root>/docker-work/moodle-docker
set -a
source .env
set +a
```

Confirm Moodle version:

```bash
grep "\$release" "$MOODLE_DOCKER_WWWROOT/version.php"
```

Run final local checks:

```bash
bin/moodle-docker-compose exec -T webserver php admin/cli/purge_caches.php
bin/moodle-docker-compose exec -T webserver php admin/cli/checks.php
bin/moodle-docker-compose exec -T webserver php admin/cli/cron.php --keep-alive=0
```

Check the local browser site before production transfer:

```text
http://localhost:8036/
```

Confirm:

- Home page loads.
- Admin login works.
- Course pages load.
- Uploaded files open.
- File upload works.
- The Moodle environment check is clean for Moodle 5.1.

## 2. Confirm Final Upgraded Local Docker Artifacts

The final DB export and production `config.php` template are already staged under `production-import`. Verify them before transfer:

```bash
UPGRADE_ROOT=<active-upgrade-root>
gzip -t "$UPGRADE_ROOT/production-import/moodle-db.sql.gz"
ls -lh "$UPGRADE_ROOT/production-import/moodle-db.sql.gz"
test -f "$UPGRADE_ROOT/production-import/config.php"
```

Confirm the final local Moodle code anchor:

```bash
cd "$UPGRADE_ROOT/docker-work/moodle"
git log -1 --oneline --decorate
git status --short
```

Expected Git anchor:

```text
17513f66142 (HEAD, tag: v5.1.6) Moodle release 5.1.6
```

Expected untracked plugin directories:

```text
public/auth/userkey/
public/availability/condition/xp/
public/blocks/checklist/
public/blocks/xp/
public/local/bilkent/
public/local/mailtest/
public/local/metagroups/
public/mod/board/
public/mod/checklist/
public/mod/game/
public/mod/hvp/
public/mod/pdfannotator/
public/mod/publication/
public/mod/questionnaire/
public/mod/ratingallocate/
public/mod/scheduler/
public/plagiarism/turnitin/
public/theme/boost_union/
public/theme/moove/
```

Confirm the 5.1 plugin source folders that will be transferred:

```bash
find "$UPGRADE_ROOT/additional-plugins/5.1/public" -maxdepth 4 -name version.php -print | sort
```

Use `additional-plugins/5.1/public` as the reviewed plugin export source for production. Do not rely on the local Moodle checkout or the local audit tarball as the plugin source; export or transfer the plugin overlay from `additional-plugins`.

Required production plugin/custom code paths:

```text
public/auth/userkey
public/availability/condition/xp
public/blocks/checklist
public/blocks/xp
public/local/bilkent
public/local/mailtest
public/local/metagroups
public/mod/board
public/mod/checklist
public/mod/game
public/mod/hvp
public/mod/pdfannotator
public/mod/publication
public/mod/questionnaire
public/mod/ratingallocate
public/mod/scheduler
public/plagiarism/turnitin
public/theme/boost_union
public/theme/moove
```

`mod_hvp` is intentionally retained because the site still has active HVP activities. Do not migrate HVP activities to core `mod_h5pactivity` or uninstall `mod_hvp` during this deployment.

The local `stage-exports/after-5.1.6/moodle-code.tgz` tarball may be kept as audit/rollback evidence, but production code should come from Git plus the plugin paths above.

If you need to regenerate the local audit tarball, use:

```bash
cd "$UPGRADE_ROOT/docker-work/moodle"
/usr/bin/tar --exclude='./.git' --exclude='.DS_Store' --exclude='._*' \
  -czf "$UPGRADE_ROOT/stage-exports/after-5.1.6/moodle-code.tgz" .
```

Verify the audit tarball only if it was regenerated:

```bash
cd "$UPGRADE_ROOT/stage-exports/after-5.1.6"
tar -tzf moodle-code.tgz | head
```

## 3. Prepare Debian 13 Trixie Production VM

Recommended production VM:

```text
Debian 13 Trixie
MariaDB 11.8
PHP 8.4 FPM
Apache or Nginx
Moodle code checked out from Git
```

On the new VM:

```bash
ssh yan@<production-moodle-host>
sudo -v
date
hostnamectl
df -hT
```

Install required packages according to the final Moodle 5.1 environment check. On Debian 13 Trixie, this normally means Apache, MariaDB 11.8, PHP 8.4 FPM, the PHP CLI, the MariaDB PHP driver, Moodle-required PHP extensions, Moodle helper binaries, Composer, and routine deployment tools such as Git, rsync, unzip, and cron.

For an Apache plus PHP-FPM production setup:

```bash
sudo apt update

sudo apt install apache2 mariadb-server git rsync unzip cron composer \
  php8.4-cli php8.4-fpm php8.4-mysql \
  php8.4-curl php8.4-gd php8.4-intl php8.4-mbstring \
  php8.4-soap php8.4-xml php8.4-zip \
  ghostscript poppler-utils aspell aspell-en graphviz python3
```

Allow the deployment user to enter group-protected Moodle directories and manage temporary deployment writes:

```bash
sudo usermod -aG www-data yan
id yan
```

If `yan` was not already a member of `www-data`, log out and reconnect before continuing. The `id` output should include `www-data`.

Enable the Apache modules and PHP-FPM configuration normally needed by Moodle:

```bash
sudo a2enmod proxy_fcgi setenvif rewrite ssl headers
sudo a2enconf php8.4-fpm
sudo systemctl restart apache2 php8.4-fpm
```

Notes:

- `apache2` is the web server package. If using Nginx instead, replace this section with an Nginx plus PHP-FPM virtual host setup.
- `php8.4-mysql` provides the MySQLi driver used for MariaDB.
- `php8.4-xml` provides `dom`, `simplexml`, `xml`, `xmlreader`, and related XML support.
- `ghostscript` provides `/usr/bin/gs`; `poppler-utils` provides `/usr/bin/pdftoppm`, which Moodle prefers for efficient PDF-to-image conversion.
- `aspell` and `aspell-en` provide `/usr/bin/aspell` and the English dictionary.
- `graphviz` provides `/usr/bin/dot`.
- `python3` provides `/usr/bin/python3` for Moodle's Python path. Do not install the `moodlemlbackend` pip package on Debian 13 unless Moodle predictive analytics are deliberately required; its current Moodle-required release depends on old Python packages that do not install cleanly on Python 3.13.
- Several checked extensions, including `json`, `openssl`, `ctype`, `hash`, `fileinfo`, `filter`, `iconv`, `pcre`, `spl`, `tokenizer`, `zlib`, and usually `sodium`, are normally provided by the base PHP packages on Debian.
- Configure `/etc/php/8.4/fpm/php.ini` for production Moodle values, especially `max_input_vars >= 5000`, plus suitable `memory_limit`, `post_max_size`, and `upload_max_filesize` values for the site.

After package installation, set these Moodle system paths:

```text
Path to ghostscript: /usr/bin/gs
Path to pdftoppm:    /usr/bin/pdftoppm
Path to aspell:      /usr/bin/aspell
Path to dot:         /usr/bin/dot
Path to Python:      /usr/bin/python3
```

Before installing Moodle code, run this readiness check. It only reports what is present or missing; it does not install or change anything.

```bash
cat >/tmp/check-moodle-51-host-readiness.sh <<'BASH'
#!/usr/bin/env bash
set -u

missing=0

check_package() {
  if dpkg-query -W -f='${Status}' "$1" 2>/dev/null | grep -q "install ok installed"; then
    printf "OK      package      %s\n" "$1"
  else
    printf "MISSING package      %s\n" "$1"
    missing=1
  fi
}

check_command() {
  if command -v "$1" >/dev/null 2>&1; then
    printf "OK      command      %-12s %s\n" "$1" "$(command -v "$1")"
  else
    printf "MISSING command      %s\n" "$1"
    missing=1
  fi
}

check_extension() {
  if php -m | grep -qi "^${1}$"; then
    printf "OK      php-ext      %s\n" "$1"
  else
    printf "MISSING php-ext      %s\n" "$1"
    missing=1
  fi
}

echo "== OS and runtime versions =="
cat /etc/debian_version 2>/dev/null || true
apache2 -v 2>/dev/null | head -1 || true
mariadb --version 2>/dev/null || mysql --version 2>/dev/null || true
php -v | head -1
composer --version 2>/dev/null || true
echo

echo "== Debian packages =="
for pkg in \
  apache2 mariadb-server git rsync unzip cron composer \
  php8.4-cli php8.4-fpm php8.4-mysql \
  php8.4-curl php8.4-gd php8.4-intl php8.4-mbstring \
  php8.4-soap php8.4-xml php8.4-zip \
  ghostscript poppler-utils aspell aspell-en graphviz python3
do
  check_package "$pkg"
done
echo

echo "== Helper commands for Moodle system paths =="
for cmd in php git rsync unzip composer mariadb aspell dot gs pdftoppm python3; do
  check_command "$cmd"
done
echo

echo "== Moodle-required PHP extensions =="
for ext in iconv mbstring curl openssl tokenizer soap ctype zip zlib gd simplexml spl pcre dom xml xmlreader intl json hash fileinfo sodium exif filter; do
  check_extension "$ext"
done
echo

echo "== Active PHP CLI settings to review =="
php -r '
$settings = ["file_uploads", "memory_limit", "max_input_vars", "post_max_size", "upload_max_filesize", "zend.exception_ignore_args"];
foreach ($settings as $setting) {
    printf("%-28s %s\n", $setting, ini_get($setting));
}
'
echo

echo "== PHP-FPM php.ini settings to review =="
if [ -f /etc/php/8.4/fpm/php.ini ]; then
  grep -E "^[[:space:]]*(file_uploads|memory_limit|max_input_vars|post_max_size|upload_max_filesize|zend.exception_ignore_args)[[:space:]]*=" /etc/php/8.4/fpm/php.ini || true
else
  echo "MISSING file        /etc/php/8.4/fpm/php.ini"
  missing=1
fi
echo

if [ "$missing" -eq 0 ]; then
  echo "READY: required packages, commands, and PHP extensions were found."
else
  echo "NOT READY: install or enable the missing items above before continuing."
fi

exit "$missing"
BASH

bash /tmp/check-moodle-51-host-readiness.sh
```

PHP settings:

The following settings are recommended:

```text
file_uploads = On
memory_limit = 512M
max_input_vars = 10000
post_max_size = 2300M
upload_max_filesize = 2048M
zend.exception_ignore_args = On

a critical setting value for the PHP (for stars integrated installations only - semester/prep): max_input_vars 1000 (default value) is not enough, i increase it to 10000 (we sent list of course/category/enrollment/users moodle web service for the sync)
```

If you want to make changes to php.ini use 
```bash
sudo nano /etc/php/8.4/fpm/php.ini
```
and after changes just restart php-fpm service with 

```bash
sudo systemctl restart php8.4-fpm.service
```
 No need to restart apache for php.ini changes.

Change the Moodle 5.x Apache document root to:

```text
/var/www/moodle/public
```

The enabled Apache config filenames may not match the Moodle hostname. For example, Certbot may have created names such as `000-default.conf` and `000-default-le-ssl.conf`. Do not rename working Apache config files just to match this runbook; list the enabled sites and identify the active HTTP and HTTPS vhost files. For this deployment, the document-root change normally only needs to be made in the SSL vhost.

```bash
sudo apache2ctl -S
sudo ls -l /etc/apache2/sites-enabled/

# Inspect the active HTTP vhost if needed, but normally only edit the SSL vhost.
sudo less /etc/apache2/sites-available/<active-http-vhost>.conf
sudo nano /etc/apache2/sites-available/<active-https-vhost>.conf

sudo apache2ctl configtest
sudo systemctl reload apache2
```

Create Moodle data directory:

```bash
sudo mkdir -p /var/moodledata
sudo chown www-data:www-data /var/moodledata
sudo chmod 2770 /var/moodledata
```

## 4. Install Moodle Code From Git

For Moodle 5.1, the web document root should point to Moodle's `public` directory, not the repository root.

Install exactly the tested Moodle core release:

```bash
cd /var/www
sudo git clone git://git.moodle.org/moodle.git moodle

cd /var/www/moodle
sudo git fetch --tags origin
sudo git checkout v5.1.6
sudo git log -1 --oneline --decorate
```

Expected result:

```text
17513f66142 (HEAD, tag: v5.1.6) Moodle release 5.1.6
```

Do not simply copy the whole old Moodle code tree into production.

Do not create `config.php` or run `admin/cli/upgrade.php` until the custom/contributed plugin code has been installed under `public/...`.

## 5. Transfer Final Artifacts To Production

From your local machine:

```bash
UPGRADE_ROOT=<active-upgrade-root>
REMOTE=yan@<production-moodle-host>
REMOTE_IMPORT=/home/yan/moodle-imports

ssh "$REMOTE" "mkdir -p '$REMOTE_IMPORT'"

rsync -av --info=progress2 \
  "$UPGRADE_ROOT/production-import/moodle-db.sql.gz" \
  "$UPGRADE_ROOT/production-import/config.php" \
  "$REMOTE:$REMOTE_IMPORT/"

rsync -a --delete --info=progress2 \
  --exclude='.DS_Store' \
  --exclude='._*' \
  "$UPGRADE_ROOT/additional-plugins/5.1/" \
  "$REMOTE:$REMOTE_IMPORT/additional-plugins-5.1/"

rsync -a --info=progress2 \
  --exclude='.DS_Store' \
  --exclude='._*' \
  --exclude='antivirus_quarantine/' \
  --exclude='cache/' \
  --exclude='localcache/' \
  --exclude='localrequest/' \
  --exclude='lost+found/' \
  --exclude='muc/' \
  --exclude='sessions/' \
  --exclude='temp/' \
  "$UPGRADE_ROOT/docker-work/moodledata/" \
  "$REMOTE:$REMOTE_IMPORT/moodledata/"
```

You may use `sftp` instead of `rsync`, but keep the same remote layout:

```text
/home/yan/moodle-imports/moodle-db.sql.gz
/home/yan/moodle-imports/config.php
/home/yan/moodle-imports/additional-plugins-5.1/public/...
/home/yan/moodle-imports/moodledata/...
```

Do not transfer `moodle-code.tgz` as the production code source. Keep that tarball only as audit/rollback evidence for the local rehearsal. Plugin/custom code should be exported from `additional-plugins/5.1/public`.

## 6. Deploy Custom And Contributed Plugin Code

On the production VM:

```bash
IMPORT_ROOT=/home/yan/moodle-imports
PLUGIN_STAGE="$IMPORT_ROOT/additional-plugins-5.1/public"
TARGET_MOODLE=/var/www/moodle

for path in \
  auth/userkey \
  availability/condition/xp \
  blocks/checklist \
  blocks/xp \
  local/bilkent \
  local/mailtest \
  local/metagroups \
  mod/board \
  mod/checklist \
  mod/game \
  mod/hvp \
  mod/pdfannotator \
  mod/publication \
  mod/questionnaire \
  mod/ratingallocate \
  mod/scheduler \
  plagiarism/turnitin \
  theme/boost_union \
  theme/moove
do
  test -f "$PLUGIN_STAGE/$path/version.php" || {
    echo "MISSING plugin source: $PLUGIN_STAGE/$path/version.php"
    exit 1
  }
done

for path in \
  auth/userkey \
  availability/condition/xp \
  blocks/checklist \
  blocks/xp \
  local/bilkent \
  local/mailtest \
  local/metagroups \
  mod/board \
  mod/checklist \
  mod/game \
  mod/hvp \
  mod/pdfannotator \
  mod/publication \
  mod/questionnaire \
  mod/ratingallocate \
  mod/scheduler \
  plagiarism/turnitin \
  theme/boost_union \
  theme/moove
do
  sudo mkdir -p "$TARGET_MOODLE/public/$(dirname "$path")"
  sudo rm -rf "$TARGET_MOODLE/public/$path"
  sudo mv "$PLUGIN_STAGE/$path" "$TARGET_MOODLE/public/$path"
  sudo chown -R www-data:www-data "$TARGET_MOODLE/public/$path"
  echo "Installed plugin code: public/$path"
done
```

Verify plugin paths and Frankenstyle components:

```bash
cd /var/www/moodle

for path in \
  public/auth/userkey \
  public/availability/condition/xp \
  public/blocks/checklist \
  public/blocks/xp \
  public/local/bilkent \
  public/local/mailtest \
  public/local/metagroups \
  public/mod/board \
  public/mod/checklist \
  public/mod/game \
  public/mod/hvp \
  public/mod/pdfannotator \
  public/mod/publication \
  public/mod/questionnaire \
  public/mod/ratingallocate \
  public/mod/scheduler \
  public/plagiarism/turnitin \
  public/theme/boost_union \
  public/theme/moove
do
  test -f "$path/version.php" && echo "OK $path/version.php" || echo "MISSING $path/version.php"
done

sudo find public/auth/userkey public/availability/condition/xp public/blocks/checklist public/blocks/xp \
  public/local/bilkent public/local/mailtest public/local/metagroups \
  public/mod/board public/mod/checklist public/mod/game public/mod/hvp public/mod/pdfannotator \
  public/mod/publication public/mod/questionnaire public/mod/ratingallocate public/mod/scheduler \
  public/plagiarism/turnitin public/theme/boost_union public/theme/moove \
  -name version.php -print -exec grep -E "\\$plugin->component" {} \;
```

Expected components:

```text
auth_userkey
availability_xp
block_checklist
block_xp
local_bilkent
local_mailtest
local_metagroups
mod_board
mod_checklist
mod_game
mod_hvp
mod_pdfannotator
mod_publication
mod_questionnaire
mod_ratingallocate
mod_scheduler
plagiarism_turnitin
theme_boost_union
theme_moove
```

## 7. Install Composer Vendor Dependencies

Moodle 5.1 expects Composer dependencies under `/var/www/moodle/vendor`. Install them after core and plugin code are in place, but before creating `config.php` and running the Moodle upgrade.

Temporarily make the code tree writable by the deployment user and readable by `www-data`:

```bash
cd /var/www/moodle
sudo chown -R yan:www-data /var/www/moodle
sudo find /var/www/moodle -type d -exec chmod 2775 {} \;
sudo find /var/www/moodle -type f -exec chmod 0664 {} \;
```

Create the Composer target directory if it is missing, then run Composer as the deployment user, not with `sudo`:

```bash
mkdir -p /var/www/moodle/vendor/composer
composer install --no-dev --classmap-authoritative
```

Expected result should include:

```text
Generating optimized autoload files
```

If Composer reports `vendor/composer/installed.json` permission errors, stop and re-check `/var/www/moodle` ownership and group-write permissions. Do not switch to `sudo composer install`.

## 8. Create Production Database And Import SQL

On the production VM:

```bash
sudo mariadb
```

Create database and database user. Replace the password before executing:

```sql
SHOW DATABASES;
SELECT User, Host FROM mysql.user;

DROP DATABASE IF EXISTS moodle;

DROP USER IF EXISTS 'moodle_user'@'localhost';

CREATE DATABASE moodle
    DEFAULT CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

CREATE USER 'moodle_user'@'localhost'
    IDENTIFIED BY 'YOUR_NEW_STRONG_PASSWORD';

GRANT ALL PRIVILEGES ON moodle.* TO 'moodle_user'@'localhost';

FLUSH PRIVILEGES;

SHOW CREATE DATABASE moodle;
SHOW GRANTS FOR 'moodle_user'@'localhost';

EXIT;
```

Import the tested upgraded database:

```bash
IMPORT_ROOT=/home/yan/moodle-imports

gzip -t "$IMPORT_ROOT/moodle-db.sql.gz"
gunzip -c "$IMPORT_ROOT/moodle-db.sql.gz" | sudo mariadb moodle
```

If the database dump already contains `CREATE DATABASE` and `USE` statements, inspect it first and adjust the import command.

Verify:

```bash
sudo mariadb moodle -e "SHOW TABLES LIKE 'mdl_config';"
sudo mariadb moodle -e "SELECT name, value FROM mdl_config WHERE name IN ('release','version');"
```

Expected release should show Moodle 5.1.6.

## 9. Install Final Moodle Data On Production

This runbook assumes Moodledata has already been uploaded to the production VM at:

```text
/home/yan/moodle-imports/moodledata
```

The upload should exclude generated runtime directories such as `cache`, `localcache`, `localrequest`, `muc`, `sessions`, and `temp`. Moodle will recreate those on the new host. Also exclude `antivirus_quarantine` unless there is a deliberate audit reason to preserve quarantined files. Exclude `lost+found` from the upload; it belongs to a filesystem root, not to Moodledata.

Persistent Moodledata content should include `filedir`. Keep `lang`, `secret`, and `trashdir` when present unless there is a deliberate reason to rebuild or discard them.

Use a move/rename instead of copying, because Moodledata is large and there may not be enough free disk space for two full copies.

On the production VM, first stop any running Moodledata `rsync` with `Ctrl-C`, then check whether the uploaded source and final target are on the same filesystem:

```bash
IMPORT_ROOT=/home/yan/moodle-imports

df -h "$IMPORT_ROOT/moodledata" /var
stat -c '%d %n' "$IMPORT_ROOT/moodledata" /var
```

If the two `stat` device numbers match, the move to `/var/moodledata` should be a rename on the same filesystem and should not duplicate the 70GB. If they differ, `mv` may copy then delete, which can still require temporary free space.

If `/var/moodledata` is only a partial failed new import directory, not a mounted filesystem root and not an active old-production Moodledata directory, remove it and rename the uploaded directory into place:

```bash
sudo rm -rf /var/moodledata
sudo mv "$IMPORT_ROOT/moodledata" /var/moodledata
```

If `/var/moodledata/lost+found` exists, treat `/var/moodledata` as a filesystem root. Do not delete `/var/moodledata` or `/var/moodledata/lost+found`; move the uploaded Moodledata contents into the existing mount point instead:

```bash
sudo rsync -a --info=progress2 "$IMPORT_ROOT/moodledata/" /var/moodledata/
```

Confirm the final layout. Moodledata contents such as `filedir`, `lang`, `secret`, and `trashdir` should be directly under `/var/moodledata`, not under `/var/moodledata/moodledata`:

```bash
sudo find /var/moodledata -mindepth 1 -maxdepth 1 -type d -print | sort | head -30
```

Reset Moodledata ownership and permissions:

```bash
sudo chown -R www-data:www-data /var/moodledata
sudo find /var/moodledata -type d -exec chmod 2770 {} \;
sudo find /var/moodledata -type f -exec chmod 0660 {} \;
```

Do not delete:

```text
/var/moodledata/filedir
/var/moodledata/lang
/var/moodledata/secret
/var/moodledata/trashdir
```

If `/var/moodledata/lost+found` exists, leave it in place as filesystem metadata. It is not part of the Moodledata transfer and should not be copied from the local rehearsal.

## 10. Create Production config.php

Create `/var/www/moodle/config.php` based on the final production paths and credentials. Do not copy the local Docker `config.php` directly.

If the reviewed production template was transferred from `production-import/config.php`, install it first and then inspect/edit the host-specific values:

```bash
IMPORT_ROOT=/home/yan/moodle-imports

sudo cp "$IMPORT_ROOT/config.php" /var/www/moodle/config.php
sudo nano /var/www/moodle/config.php
```

Confirm the template has the correct production database password, `wwwroot`, `dataroot`, and any reverse proxy settings before running Moodle CLI checks.

Critical values:

```php
<?php
unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodle_user';
$CFG->dbpass    = 'YOUR_NEW_STRONG_PASSWORD';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = [
    'dbpersist' => false,
    'dbport' => '',
    'dbsocket' => '',
    'dbcollation' => 'utf8mb4_unicode_ci',
];

$CFG->wwwroot   = 'https://<production-moodle-host>';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';
$CFG->directorypermissions = 02770;

require_once(__DIR__ . '/lib/setup.php');
```

Directories are accessible to owner/group, not to everyone else, and the setgid bit helps new directories inherit the group.

Do not reuse the old `wwwroot` unless the new VM is taking over the same hostname.

Optional production values, if appropriate for this site:

```php
$CFG->sslproxy = true;
$CFG->reverseproxy = true;
$CFG->pathtophp = '/usr/bin/php';
$CFG->noemailever = false;
```

Lock `config.php` permissions:

```bash
sudo chown root:www-data /var/www/moodle/config.php
sudo chmod 0640 /var/www/moodle/config.php
```

For Moodle 5.1, the web server document root must be:

```text
/var/www/moodle/public
```

## 11. Run Production Moodle Checks

On the production VM:

```bash
cd /var/www/moodle
sudo -u www-data php admin/cli/purge_caches.php
sudo -u www-data php admin/cli/checks.php
sudo -u www-data php admin/cli/upgrade.php --non-interactive
sudo -u www-data php admin/cli/purge_caches.php
sudo -u www-data php admin/cli/checks.php
sudo -u www-data php admin/cli/cron.php --keep-alive=0
```

If `checks.php`, `upgrade.php`, cron, or browser smoke testing reports a fatal error, stop and diagnose before enabling cron or cutover.

Check in the browser:

- Home page loads.
- Admin login works.
- Course pages load.
- Uploaded files open.
- File upload works.
- HVP activities load.
- Checklist activities load.
- PDF Annotator activities load.
- Questionnaire activities load.
- Scheduler activities load.
- Turnitin settings and representative Turnitin assignments load.
- Email works.
- Scheduled tasks are not failing repeatedly.
- Moodle environment check is clean for Moodle 5.1.

## 12. Finalize Production Ownership And Permissions

After the Moodle upgrade, CLI checks, and browser smoke tests are successful, lock production code back down so the web server can read it but cannot write it:

```bash
sudo chown -R root:www-data /var/www/moodle
sudo find /var/www/moodle -type d -exec chmod 2750 {} \;
sudo find /var/www/moodle -type f -exec chmod 0640 {} \;
sudo chown root:www-data /var/www/moodle/config.php
sudo chmod 0640 /var/www/moodle/config.php
```

Keep Moodledata owned by the web server user/group with group inheritance:

```bash
sudo chown -R www-data:www-data /var/moodledata
sudo find /var/moodledata -type d -exec chmod 2770 {} \;
sudo find /var/moodledata -type f -exec chmod 0660 {} \;
```

Verify the deployment user remains in `www-data` for future maintenance access:

```bash
id yan
ls -ld /var/www/moodle /var/www/moodle/vendor /var/moodledata
```

Expected ownership/permission shape:

```text
/var/www/moodle      root:www-data, directories 2750, files 0640
/var/moodledata      www-data:www-data, directories 2770, files 0660
yan                  member of www-data
```

## 13. Enable Production Cron

Only after CLI and browser checks pass:

Enable and verify the system cron service:

```bash
sudo systemctl enable --now cron
sudo systemctl status cron
```

Enable Moodle's internal cron switch. This is separate from the system cron daemon and the `www-data` crontab:

```bash
cd /var/www/moodle
sudo -u www-data /usr/bin/php admin/cli/cron.php --enable
sudo -u www-data /usr/bin/php admin/cli/cfg.php --name=cron_keepalive --set=50
sudo -u www-data /usr/bin/php admin/cli/cfg.php --name=cron_keepalive
```

```bash
sudo crontab -u www-data -e
```

Use the production cron entry:

```cron
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php >/dev/null 2>&1
```

Do not use `/var/www/html/admin/cli/cron.php`. In this deployment, Apache serves `/var/www/moodle/public`, but Moodle CLI scripts remain under `/var/www/moodle/admin/cli`.

Then verify one manual cron run:

```bash
sudo -u www-data /usr/bin/php /var/www/moodle/admin/cli/cron.php --keep-alive=50
sudo -u www-data /usr/bin/php /var/www/moodle/admin/cli/checks.php
```

Confirm the saved `www-data` crontab points at `/var/www/moodle`, not `/var/www/html`:

```bash
sudo crontab -u www-data -l
sudo crontab -u www-data -l | grep '/var/www/moodle/admin/cli/cron.php'
sudo crontab -u www-data -l | grep -q '/var/www/html/admin/cli/cron.php' && echo 'STOP: cron points at old /var/www/html path' || echo 'OK no old /var/www/html cron path'
```

If `checks.php` reports `WARNING: Cron running (tool_task_cronrunning)`, confirm whether cron is actually firing before changing the schedule:

```bash
sudo crontab -u www-data -l
sudo systemctl status cron
sudo -u www-data php /var/www/moodle/admin/cli/cron.php --list
```

The warning can appear when cron is started every minute but completed runs are several minutes apart because long tasks or Moodle's keepalive setting keep processes alive. For a normal once-per-minute system cron setup, keep the crontab entry simple and set Moodle's admin keepalive value to `50` seconds:

```text
Site administration > Server > Tasks > Task processing > Keep alive time
```

Confirm the setting from CLI:

```bash
cd /var/www/moodle
sudo -u www-data /usr/bin/php admin/cli/cfg.php --name=cron_keepalive
sudo -u www-data /usr/bin/php admin/cli/checks.php
```

## 14. Cutover Notes

For final cutover, take a fresh old-production export during a maintenance window and repeat the tested upgrade process.

Recommended final cutover flow:

1. Put old source site in maintenance mode.
2. Stop old Moodle cron.
3. Export a fresh source database and fresh Moodle data delta.
4. Re-run the tested local upgrade process, or replay the tested upgrade procedure on the prepared production VM.
5. Transfer the final upgraded database, 5.1 plugin source, and Moodledata.
6. Install plugin code and run `composer install --no-dev --classmap-authoritative` from `/var/www/moodle`.
7. Import final upgraded database and Moodle data to production.
8. Run Moodle CLI checks and browser smoke tests.
9. Lock final code and Moodledata permissions.
10. Switch DNS or load balancer to the new VM.
11. Enable production cron.
12. Keep the old VM unchanged until the new site is accepted.

## Production Import Summary Template

```text
New production host:
Production OS:
Production PHP version:
Production MariaDB version:
Production Moodle code path:
Production Moodle data path:
Production web document root:
Production database name:
Production database user:

Final Moodle version:
Final DB import file:
Final Moodle data import source:
Final plugin list verified:
Composer vendor installed:
Code ownership verified:
Moodledata ownership verified:
Cron enabled:
DNS/load balancer switched:
Old VM retained:
```
