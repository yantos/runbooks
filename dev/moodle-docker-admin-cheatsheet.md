# Admin Cheatsheet For Moodle Docker

Frequently used local Docker administration commands during a Moodle upgrade rehearsal.

## Moodle Health Block

Run after every runtime, code, plugin, or DB change.

For Moodle 4.5 and later: (n.b, Moodle 3.11 has no --keep-alive=0 flag for cron)

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T webserver php admin/cli/purge_caches.php
bin/moodle-docker-compose exec -T webserver php admin/cli/check_database_schema.php
bin/moodle-docker-compose exec -T webserver php admin/cli/cron.php --keep-alive=0
bin/moodle-docker-compose exec -T webserver ps aux | grep -E '[a]dmin/cli/cron.php|[a]dmin/tool/task/cli/adhoc_task.php'
bin/moodle-docker-compose exec -T webserver php admin/cli/checks.php
```

Check that no ad hoc tasks are stuck after health runs.

## Maintenance Mode

CLI maintenance mode uses a file in `moodledata`, so `mdl_config.maintenance_enabled` can still show `0`.

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T webserver php admin/cli/maintenance.php
bin/moodle-docker-compose exec -T webserver php admin/cli/maintenance.php --enable
bin/moodle-docker-compose exec -T webserver php admin/cli/maintenance.php --disable
bin/moodle-docker-compose exec -T webserver sh -lc 'ls -l /var/www/moodledata/climaintenance.html 2>/dev/null || echo "no CLI maintenance file"'
```

If maintenance mode blocks cron in Docker, either skip cron until the end of the stage or temporarily disable it:

```bash
bin/moodle-docker-compose exec -T webserver php admin/cli/maintenance.php --disable
bin/moodle-docker-compose exec -T webserver php admin/cli/cron.php --keep-alive=0
bin/moodle-docker-compose exec -T webserver php admin/cli/checks.php
bin/moodle-docker-compose exec -T webserver php admin/cli/maintenance.php --enable
```

## Cache Rescue

Use after moving backward between Moodle code versions, or after stale MUC/cache errors:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T webserver sh -lc '
rm -f /var/www/moodledata/muc/config.php
rm -rf /var/www/moodledata/cache/*
rm -rf /var/www/moodledata/localcache/*
rm -rf /var/www/moodledata/temp/*
'
```

Then run the Moodle health block.

## Database Snapshot

Safe dump pattern for MariaDB 10.11 and earlier images where `mysqldump` exists:

```bash
cd "$MOODLE_DOCKER_DIR"

DUMP_DIR="$UPGRADE_ROOT/stage-exports/db-snapshot-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$DUMP_DIR"

bin/moodle-docker-compose exec -T db sh -c \
  'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction --quick --routines --triggers --events "$MYSQL_DATABASE" | gzip -1 > /tmp/moodle-db.sql.gz'

DB_CONTAINER=$(bin/moodle-docker-compose ps -q db | /usr/bin/tail -n 1)
docker cp "$DB_CONTAINER:/tmp/moodle-db.sql.gz" "$DUMP_DIR/moodle-db.sql.gz"

gzip -t "$DUMP_DIR/moodle-db.sql.gz"
ls -lh "$DUMP_DIR/moodle-db.sql.gz"
```

Safe dump pattern for MariaDB 11.8 images where the old `mysql`/`mysqldump` names may not exist:

```bash
cd "$MOODLE_DOCKER_DIR"

DUMP_DIR="$UPGRADE_ROOT/stage-exports/db-snapshot-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$DUMP_DIR"

bin/moodle-docker-compose exec -T db sh -c \
  'mariadb-dump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction --quick --routines --triggers --events "$MYSQL_DATABASE" | gzip -1 > /tmp/moodle-db.sql.gz'

DB_CONTAINER=$(bin/moodle-docker-compose ps -q db | /usr/bin/tail -n 1)
docker cp "$DB_CONTAINER:/tmp/moodle-db.sql.gz" "$DUMP_DIR/moodle-db.sql.gz"

gzip -t "$DUMP_DIR/moodle-db.sql.gz"
ls -lh "$DUMP_DIR/moodle-db.sql.gz"
```

Avoid piping `bin/moodle-docker-compose exec ... dump` directly into a local file. The Moodle Docker wrapper can print local option text into stdout and contaminate the SQL file.

## Restore A DB Snapshot

Use this when the local Docker database needs to return to a saved stage snapshot. This destroys and recreates only the Docker DB volume for the current `COMPOSE_PROJECT_NAME`; it does not touch `source-export`, `additional-plugins`, `stage-exports`, Moodle code, or Moodledata bind mounts.

Confirm you are in the intended rehearsal's `moodle-docker` directory and that `.env` names the intended project before running `down -v`:

```bash
cd "$MOODLE_DOCKER_DIR"

printf 'COMPOSE_PROJECT_NAME=%s\nMOODLE_DOCKER_DB_VERSION=%s\nMOODLE_DOCKER_PHP_VERSION=%s\nMOODLE_DOCKER_WEB_PORT=%s\n' \
  "$COMPOSE_PROJECT_NAME" "$MOODLE_DOCKER_DB_VERSION" "$MOODLE_DOCKER_PHP_VERSION" "$MOODLE_DOCKER_WEB_PORT"
```

Restore a gzipped snapshot:

```bash
cd "$MOODLE_DOCKER_DIR"

SNAPSHOT_SQL="$UPGRADE_ROOT/stage-exports/<snapshot-name>/moodle-db.sql.gz"
gzip -t "$SNAPSHOT_SQL"

bin/moodle-docker-compose down -v
bin/moodle-docker-compose up -d db
bin/moodle-docker-wait-for-db

DB_CONTAINER=$(bin/moodle-docker-compose ps -q db | /usr/bin/tail -n 1)
docker cp "$SNAPSHOT_SQL" "$DB_CONTAINER:/tmp/moodle-db.sql.gz"

bin/moodle-docker-compose exec -T db sh -lc \
  "mysql -uroot -p\"\$MYSQL_ROOT_PASSWORD\" -e 'CREATE DATABASE IF NOT EXISTS $MOODLE_DOCKER_DBNAME DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci'"

bin/moodle-docker-compose exec -T db sh -lc \
  "gzip -dc /tmp/moodle-db.sql.gz | mysql -uroot -p\"\$MYSQL_ROOT_PASSWORD\" \"$MOODLE_DOCKER_DBNAME\""

bin/moodle-docker-compose up -d webserver
bin/moodle-docker-compose ps
```

After restore, run the stage health checks before continuing the upgrade.

## Database Query Blocks

Use `mysql` for MariaDB 10.11 and earlier images:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T db mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MOODLE_DOCKER_DBNAME" -e "
SELECT name, value
FROM mdl_config
WHERE name IN ('release','version','theme','themelist','maintenance_enabled');
"
```

Use `mariadb` for MariaDB 11.8 images:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T db mariadb -uroot -p"$MYSQL_ROOT_PASSWORD" "$MOODLE_DOCKER_DBNAME" -e "
SELECT name, value
FROM mdl_config
WHERE name IN ('release','version','theme','themelist','maintenance_enabled');
"
```

To run any DB query in Docker:

```bash
bin/moodle-docker-compose exec -T db mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MOODLE_DOCKER_DBNAME" -e "SQL_QUERY"
bin/moodle-docker-compose exec -T db mariadb -uroot -p"$MYSQL_ROOT_PASSWORD" "$MOODLE_DOCKER_DBNAME" -e "SQL_QUERY"
```

To open an interactive DB prompt, do not use `-T`. The `-T` flag disables the TTY and is for one-off commands:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec db mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MOODLE_DOCKER_DBNAME"
bin/moodle-docker-compose exec db mariadb -uroot -p"$MYSQL_ROOT_PASSWORD" "$MOODLE_DOCKER_DBNAME"
```

## Runtime Checks

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T webserver php -v
bin/moodle-docker-compose exec -T webserver php -m | /usr/bin/grep -Ei 'sodium|intl|mysqli|zip|curl|mbstring|xml'
bin/moodle-docker-compose exec -T webserver php -i | /usr/bin/grep -E 'max_input_vars|memory_limit|opcache.enable'
bin/moodle-docker-compose exec -T db mysql --version || bin/moodle-docker-compose exec -T db mariadb --version
```

## Database Normalization

Collation check and conversion:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T db mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MOODLE_DOCKER_DBNAME" -e "
SELECT table_collation, COUNT(*) AS tables
FROM information_schema.tables
WHERE table_schema = '$MOODLE_DOCKER_DBNAME'
GROUP BY table_collation
ORDER BY tables DESC, table_collation;
"

bin/moodle-docker-compose exec -T webserver php admin/cli/mysql_collation.php --collation=utf8mb4_unicode_ci
bin/moodle-docker-compose exec -T webserver php admin/cli/check_database_schema.php
```

Engine and compressed row checks:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose exec -T webserver php admin/cli/mysql_engine.php --list
bin/moodle-docker-compose exec -T webserver php admin/cli/mysql_compressed_rows.php --list
bin/moodle-docker-compose exec -T webserver php admin/cli/mysql_compressed_rows.php --fix
```

Compressed rows listed as `Compressed` are not bad by themselves. Worry only if Moodle reports `Compact` or `Redundant` rows that need fixing.

## Browser Baseline

Open these after each major stage:

```text
http://localhost:8036/
http://localhost:8036/admin/index.php
http://localhost:8036/admin/plugins.php
http://localhost:8036/my/
```

Check:

- Front page loads.
- Login/admin session works.
- Dashboard loads.
- Plugins overview loads.
- A normal course page loads.
- Representative plugin activities load.
- Uploaded files download.
- No obvious PHP warnings or directory listings appear.

## Optional Trash Cleanup

Do not include this in routine health checks. It permanently empties Moodle's file trash and may remove files users still expect to recover.

Only run it when you have deliberately decided to clear recoverable deleted files:

```bash
cd "$MOODLE_DOCKER_DIR"
bin/moodle-docker-compose exec -T webserver php admin/cli/scheduled_task.php --execute='\core\task\file_trash_cleanup_task'
```
