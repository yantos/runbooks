# Moodle Data Privacy Requests: Repair `user id = 0` References

Run this outside working hours. Put Moodle into maintenance mode first, and take it out of maintenance mode only after the data requests screen has been checked.

This runbook repairs data privacy request rows where `requestedby` or `usermodified` is `0`. On the replica, this caused Moodle's data requests admin screen to fail with:

```text
dml_missing_record_exception
Invalid user (SELECT * FROM {user} WHERE id = ? [array ( 0 => 0, )])
```

The tested fix is to replace only those `0` references with a real admin user id. Leave `dpo = 0` alone unless a separate error specifically implicates it.

## Assumptions

- Production is a plain Debian 13 VM.
- You have `sudo` access.
- Moodle is installed at `/var/www/html`.
- The Moodle database tables use the default `mdl_` prefix.
- You will replace placeholders before running commands.

## Placeholders

Set these for your environment:

```bash
MOODLE_DIR="/var/www/moodle"
DB_NAME="moodle"
ADMIN_ID=2
```

## 1. Confirm Moodle Database Settings

```bash
sudo grep -E "db(user|pass|name|host|type|prefix)" "$MOODLE_DIR/config.php"
```

Confirm:

- Database name
- Table prefix, expected to be `mdl_`
- Database type, expected to be MariaDB/MySQL-compatible

If the table prefix is not `mdl_`, update every SQL command below accordingly.

## 2. Put Moodle In Maintenance Mode

Do this before changing the database.

```bash
cd "$MOODLE_DIR"
sudo -u www-data /usr/bin/php admin/cli/maintenance.php --enable
```

Optional check:

```bash
cd "$MOODLE_DIR"
sudo -u www-data /usr/bin/php admin/cli/maintenance.php
```

## 3. Take A Full Database Backup

```bash
sudo mariadb-dump "$DB_NAME" > "$HOME/prod_before_dataprivacy_zero_userrefs_$(date +%Y%m%d_%H%M%S).sql"
```

Confirm the backup exists and is non-empty:

```bash
ls -lh "$HOME"/prod_before_dataprivacy_zero_userrefs_*.sql
```

## 4. Check The Current Problem Counts

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SELECT
  SUM(requestedby = 0) AS requestedby_zero,
  SUM(usermodified = 0) AS usermodified_zero,
  SUM(dpo = 0) AS dpo_zero,
  SUM(requestedby = 0 AND usermodified = 0) AS both_zero
FROM mdl_tool_dataprivacy_request;
"
```

Expected from replica testing:

- `requestedby_zero`: about 4140
- `usermodified_zero`: about 4155
- `dpo_zero`: about 4226

Small differences are possible if production changed after the replica was copied.

## 5. Confirm The Admin User ID

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SELECT id, username, email, deleted, suspended
FROM mdl_user
WHERE deleted = 0
  AND username = 'admin';
"
```

Set `ADMIN_ID` to the returned `id`.

Example:

```bash
ADMIN_ID="2"
```

Do not continue until this is confirmed.

## 6. Export The Affected Rows

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SELECT
  id,
  userid,
  requestedby,
  dpo,
  usermodified,
  type,
  status,
  creationmethod,
  FROM_UNIXTIME(timecreated) AS request_created,
  FROM_UNIXTIME(timemodified) AS request_modified,
  LEFT(comments, 200) AS comments_preview
FROM mdl_tool_dataprivacy_request
WHERE requestedby = 0
   OR usermodified = 0
ORDER BY status, timemodified;
" > "$HOME/prod_zero_userrefs_export_$(date +%Y%m%d_%H%M%S).tsv"
```

Confirm the export exists:

```bash
ls -lh "$HOME"/prod_zero_userrefs_export_*.tsv
```

## 7. Create An In-Database Backup Table

Use today's date in the table name. The example below uses `20260902`.

```bash
sudo mariadb "$DB_NAME" -e "
CREATE TABLE z_dataprivacy_zero_userrefs_backup_20260903 AS
SELECT *
FROM mdl_tool_dataprivacy_request
WHERE requestedby = 0
   OR usermodified = 0;
"
```

Confirm the backup table count:

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SELECT COUNT(*) AS backed_up
FROM z_dataprivacy_zero_userrefs_backup_20260903;
"
```

The count should match the number of distinct affected request rows.

## 8. Apply The Fix

This updates only `requestedby` and `usermodified` values that are currently `0`.

```bash
sudo mariadb "$DB_NAME" -e "
UPDATE mdl_tool_dataprivacy_request
SET
  requestedby = CASE WHEN requestedby = 0 THEN $ADMIN_ID ELSE requestedby END,
  usermodified = CASE WHEN usermodified = 0 THEN $ADMIN_ID ELSE usermodified END
WHERE requestedby = 0
   OR usermodified = 0;

SELECT ROW_COUNT() AS rows_changed;
"
```

Do not update `dpo = 0` as part of this fix.

## 9. Verify The Fix

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SELECT
  SUM(requestedby = 0) AS requestedby_zero,
  SUM(usermodified = 0) AS usermodified_zero
FROM mdl_tool_dataprivacy_request;
"
```

Expected:

```text
requestedby_zero    0
usermodified_zero   0
```

Then reload the Moodle data requests admin screen and confirm it loads.

## 10. Take Moodle Out Of Maintenance Mode

Only do this after the data requests screen has loaded successfully.

```bash
cd "$MOODLE_DIR"
sudo -u www-data /usr/bin/php admin/cli/maintenance.php --disable
```

Optional check:

```bash
cd "$MOODLE_DIR"
sudo -u www-data /usr/bin/php admin/cli/maintenance.php
```

## 11. Dump And Drop The Temporary Backup Table

After the data requests screen has loaded successfully and Moodle is back out of maintenance mode, dump the temporary in-database backup table separately and then remove it from production.

Dump the backup table:

```bash
sudo mariadb-dump "$DB_NAME" z_dataprivacy_zero_userrefs_backup_20260903 \
  > "$HOME/z_dataprivacy_zero_userrefs_backup_20260903_$(date +%Y%m%d_%H%M%S).sql"
```

Confirm the dump exists and is non-empty:

```bash
ls -lh "$HOME"/z_dataprivacy_zero_userrefs_backup_20260903_*.sql
```

Optional row-count check before dropping:

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SELECT COUNT(*) AS backed_up_rows
FROM z_dataprivacy_zero_userrefs_backup_20260903;
"
```

Drop the temporary table:

```bash
sudo mariadb "$DB_NAME" -e "
DROP TABLE z_dataprivacy_zero_userrefs_backup_20260903;
"
```

Confirm it is gone:

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SHOW TABLES LIKE 'z_dataprivacy_zero_userrefs_backup_20260903';
"
```

If this command shows only a header like this, and no row underneath it, the table is gone:

```text
Tables_in_moodle (z_dataprivacy_zero_userrefs_backup_20260903)
```

No Moodle cache purge is needed for this cleanup step.

## Rollback

Use rollback only if the screen fails or another issue appears immediately after the update.

This restores the two repaired columns from the in-database backup table:

```bash
sudo mariadb "$DB_NAME" -e "
UPDATE mdl_tool_dataprivacy_request dr
JOIN z_dataprivacy_zero_userrefs_backup_20260903 b ON b.id = dr.id
SET
  dr.requestedby = b.requestedby,
  dr.usermodified = b.usermodified;
"
```

Verify rollback:

```bash
sudo mariadb "$DB_NAME" --batch --raw --quick -e "
SELECT
  SUM(requestedby = 0) AS requestedby_zero,
  SUM(usermodified = 0) AS usermodified_zero
FROM mdl_tool_dataprivacy_request;
"
```

Then decide whether to keep Moodle in maintenance mode while investigating.

## Notes

- The replica test confirmed that repairing `requestedby = 0` and `usermodified = 0` allowed the data requests admin screen to load again.
- `dpo = 0` was present but was not required to fix the screen load failure.
- Separately, thousands of active deletion requests target already-soft-deleted users whose deleted Moodle `username` contains `@rambler.ru`. That is a cleanup issue after the admin screen is restored, not the immediate screen-loading fix.
