# Moodle 5.1.x Safe Git Upgrade Runbook

This runbook upgrades a production Moodle installation between patch releases using an official Git tag, while preserving locally installed, untracked plugins and the controlled `restorefile.php` archive-download patch. The example upgrades `v5.1.6` to `v5.1.7` in `/var/www/moodle`.

## Assumptions

- Moodle's Git repository is `/var/www/moodle`.
- The current release is checked out at the official tag `v5.1.6`.
- The target release is the official tag `v5.1.7`.
- Locally installed plugins are untracked or ignored by Git.
- Moodle CLI scripts are under `/var/www/moodle/admin/cli`.
- The archive-download patch helper is available outside the Moodle checkout. Replace `/path/to/moodle_scripts` below with its actual location.
- The web-server account is `www-data`. Replace it if the server uses a different account.
- Commands requiring elevated privileges are run with `sudo`.

Set a maintenance window before starting. Do not perform the production upgrade until the same procedure has succeeded on a representative test site.

## 1. Confirm the Current State

```bash
cd /var/www/moodle
sudo git status
sudo git describe --tags --exact-match HEAD
sudo git remote -v
```

Expected result: `HEAD` is at `v5.1.6`. The only acceptable tracked modification is `public/backup/restorefile.php` containing the controlled MDL-63645 archive-download patch. Untracked plugin directories are acceptable if they are known and intentional.

Inspect the tracked modification and confirm the patch marker is present:

```bash
sudo git diff -- public/backup/restorefile.php
sudo grep -nF 'Local patch for MDL-63645' public/backup/restorefile.php
test "$(sudo git diff --name-only HEAD)" = "public/backup/restorefile.php"
```

Stop if another tracked file is modified, the marker is absent, or the diff differs from the reviewed capability check.

The installation uses `.git/info/exclude` for the following local plugins and themes:

```text
/public/auth/userkey/
/public/local/bilkent/
/public/local/mailtest/
/public/mod/board/
/public/mod/checklist/
/public/mod/game/
/public/mod/hvp/
/public/mod/pdfannotator/
/public/mod/publication/
/public/mod/questionnaire/
/public/mod/ratingallocate/
/public/mod/scheduler/
/public/plagiarism/turnitin/
/public/theme/boost_union/
/public/theme/moove/
```

Review both the rules and the files they hide:

```bash
sudo cat "$(sudo git rev-parse --git-path info/exclude)"
sudo git status --ignored
sudo git ls-files --others --ignored --exclude-standard --directory
```

To identify the rule that ignores a particular plugin:

```bash
sudo git check-ignore -v public/mod/hvp/
```

Record the installed plugin list and confirm that every plugin and theme is compatible with the target Moodle release. A core Git checkout neither updates nor removes third-party plugins.

Check for source backups left by an older version of the archive patch helper. No PHP source backup should remain under the web root:

```bash
sudo find public/backup -maxdepth 1 -type f -name 'restorefile.php*.bak' -print
```

If this prints anything, record the paths. After completing the code backup in section 2, move those files outside the Moodle web root before checking out the new tag.

## 2. Prepare Recoverable Backups

Create and verify backups of all three parts of the site:

1. The complete Moodle code directory, including untracked plugins and `config.php`.
2. The complete Moodle data directory (`moodledata`).
3. A consistent database dump.

Record the current commit for reference:

```bash
cd /var/www/moodle
sudo git rev-parse HEAD
```

Confirm that the backups are readable and that there is enough free disk space for the upgrade. A Git checkout is not a substitute for a code backup because Git does not contain the untracked plugins.

## 3. Fetch and Inspect the Target Tag

```bash
cd /var/www/moodle
sudo git fetch --tags --prune origin
sudo git tag --list v5.1.7
sudo git log -1 --oneline v5.1.7
```

The tag command must print `v5.1.7`. Stop if the tag is absent or the remote is not the expected official Moodle repository.

Check whether the target release contains tracked files at any local plugin path. Add every locally installed, untracked plugin path to this command:

```bash
sudo git ls-tree -r --name-only v5.1.7 -- \
  public/auth/userkey \
  public/local/bilkent \
  public/local/mailtest \
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
```

Expected result: no output. If Git reports files under one of these paths, stop and resolve that path before checkout. Git normally refuses to overwrite an untracked file, but this preflight makes the conflict visible in advance.

## 4. Enter the Maintenance Window

Enable maintenance mode:

```bash
cd /var/www/moodle
sudo -u www-data php admin/cli/maintenance.php --enable
```

Stop or pause Moodle cron and any queue workers using the server's normal service-management procedure. If the site can still receive writes through integrations, disable those entry points too.

After writes have stopped, take the final production database backup. This is the database backup to use for rollback.

## 5. Install the New Core Files

Perform one final status check. Restore the patched tracked file from the current tag so that checkout starts from pristine core, then check out the release tag:

```bash
cd /var/www/moodle
sudo git status
sudo git diff -- public/backup/restorefile.php
sudo git restore --source=HEAD -- public/backup/restorefile.php
test -z "$(sudo git status --short --untracked-files=no)"
sudo git checkout v5.1.7
sudo git describe --tags --exact-match HEAD
```

The final command must print `v5.1.7`. A detached `HEAD` is normal when an official release tag is checked out.

Reapply the archive-download patch to the new core while the site remains in maintenance mode. Always run the dry-run first. If it does not recognize the expected upstream code, stop and review whether the target release changed or fixed MDL-63645; do not force the old replacement onto changed code.

```bash
PATCH_HELPER=/path/to/moodle_scripts/archive-admin/patch_restorefile_access.php

sudo php "$PATCH_HELPER" /var/www/moodle
sudo php "$PATCH_HELPER" --apply /var/www/moodle
sudo php -l public/backup/restorefile.php
sudo git diff --check
test "$(sudo git diff --name-only HEAD)" = "public/backup/restorefile.php"
sudo git diff -- public/backup/restorefile.php
```

The helper creates no backup or other untracked file in the Moodle checkout. The final diff must contain only the reviewed capability check in `public/backup/restorefile.php`.

Do NOT run `git clean -fd` or `git clean -fdx`. Those commands can delete the local plugins; `-x` also includes ignored files.

Recheck the local plugins before continuing:

```bash
sudo git status --ignored
sudo git ls-files --others --ignored --exclude-standard --directory
```

## 6. Run the Moodle Upgrade

Run the database upgrade as the web-server account:

```bash
cd /var/www/moodle
sudo -u www-data php admin/cli/upgrade.php
```

Read the output and stop to investigate any error. Do not disable maintenance mode after a failed or incomplete database upgrade.

When the upgrade completes successfully, purge caches:

```bash
sudo -u www-data php admin/cli/purge_caches.php
```

## 7. Verify Before Reopening

While the site is still in maintenance mode:

```bash
cd /var/www/moodle
sudo git describe --tags --exact-match HEAD
sudo git status --ignored
sudo git diff --check
test "$(sudo git diff --name-only HEAD)" = "public/backup/restorefile.php"
sudo git diff -- public/backup/restorefile.php
```

Also verify:

- The Moodle version shown by the CLI or administration interface is `5.1.7`.
- The expected plugins and themes are present.
- File ownership and permissions are unchanged and appropriate.
- Web-server and PHP logs contain no new fatal errors.
- A user with the archived teacher role can open the backup file page and download a private backup.
- The archived teacher cannot upload a backup and receives an access-control error after selecting Restore.
- An administrator can still use the normal restore workflow.
- The affected gradebook calculation works correctly on a controlled test course.
- Login, course access, scheduled tasks, email, and critical integrations work.

## 8. Reopen the Site

Disable maintenance mode only after verification succeeds:

```bash
cd /var/www/moodle
sudo -u www-data php admin/cli/maintenance.php --disable
```

Restart Moodle cron and any queue workers. Monitor application, PHP, database, and web-server logs during the first production use, then confirm scheduled tasks are running normally.

## Rollback

Do not roll back only the Git checkout after the Moodle database upgrade has run. The old code may be incompatible with the upgraded database.

For a full rollback:

1. Re-enable maintenance mode and stop cron/workers.
2. Restore the pre-upgrade database backup.
3. Restore the complete pre-upgrade code backup, including plugins, `config.php`, and the reviewed `restorefile.php` patch. If restoring pristine Git code instead, run the patch helper in dry-run mode and reapply the patch appropriate to that release.
4. Restore `moodledata` if the failed upgrade changed it or if the backup plan requires the three components to be restored as a matched set.
5. Purge caches, verify the restored version, and test the site before reopening.

Keep the maintenance window open and escalate the failure if any backup cannot be restored cleanly.
