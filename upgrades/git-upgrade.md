# Moodle 5.1.x Safe Git Upgrade Runbook

This runbook upgrades a production Moodle installation between patch releases using an official Git tag, while preserving locally installed, untracked plugins. The example upgrades `v5.1.6` to `v5.1.7` in `/var/www/moodle`.

## Assumptions

- Moodle's Git repository is `/var/www/moodle`.
- The current release is checked out at the official tag `v5.1.6`.
- The target release is the official tag `v5.1.7`.
- Locally installed plugins are untracked or ignored by Git.
- Moodle CLI scripts are under `/var/www/moodle/admin/cli`.
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

Expected result: `HEAD` is at `v5.1.6`, and there are no modified or staged tracked files. Untracked plugin directories are acceptable if they are known and intentional.

If custom plugins appear are not ignored (i.e., they appear in git status), you may add them to .git/info/exclude. This is not necessary, as git will ignore anything not tracked during the update, but it does help to keep things organized:

```bash
sudo tee -a .git/info/exclude >/dev/null <<'EOF'
/public/auth/userkey/
/public/local/bilkent/
/public/local/mailtest/
/public/local/metagroups/
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
EOF
```
If plugins are listed in `.git/info/exclude`, review both the rules and the files they hide:

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

Perform one final status check, then check out the release tag:

```bash
cd /var/www/moodle
sudo git status
sudo git checkout v5.1.7
sudo git describe --tags --exact-match HEAD
```

The final command must print `v5.1.7`. A detached `HEAD` is normal when an official release tag is checked out.

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
```

Also verify:

- The Moodle version shown by the CLI or administration interface is `5.1.7`.
- The expected plugins and themes are present.
- File ownership and permissions are unchanged and appropriate.
- Web-server and PHP logs contain no new fatal errors.
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
3. Restore the complete pre-upgrade code backup, including plugins and `config.php`.
4. Restore `moodledata` if the failed upgrade changed it or if the backup plan requires the three components to be restored as a matched set.
5. Purge caches, verify the restored version, and test the site before reopening.

Keep the maintenance window open and escalate the failure if any backup cannot be restored cleanly.