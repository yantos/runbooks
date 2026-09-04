# Moodle Docker Frequent Commands

Common local Docker commands for a Moodle upgrade rehearsal.

## Docker Basics

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose ps
bin/moodle-docker-compose up -d
bin/moodle-docker-compose restart
bin/moodle-docker-compose stop
bin/moodle-docker-compose logs --tail=200 webserver
bin/moodle-docker-compose logs --tail=200 db
```

Do not use `down -v` unless intentionally destroying the local DB volume and rebuilding from a snapshot.

## Safe Stop And Restart

Preserves the current DB volume:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose stop
bin/moodle-docker-compose start
bin/moodle-docker-compose ps
```

## Recreate Webserver

Use after changing `.env`, images, `local.yml`, or Apache config. Preserves the current DB volume:

```bash
cd mainmoodle-upgrade-2026/docker-work/moodle-docker

set -a
source .env
set +a

cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose stop webserver
bin/moodle-docker-compose pull webserver
bin/moodle-docker-compose up -d --force-recreate webserver
bin/moodle-docker-compose ps
```

## Recreate Database Container

Use after changing `MOODLE_DOCKER_DB_VERSION`. Preserves the named DB volume:

```bash
cd "$MOODLE_DOCKER_DIR"
set -a
source .env
set +a

bin/moodle-docker-compose stop webserver
bin/moodle-docker-compose stop db
bin/moodle-docker-compose pull db
bin/moodle-docker-compose up -d db
bin/moodle-docker-wait-for-db

bin/moodle-docker-compose exec -T db sh -lc \
  'mariadb-upgrade -uroot -p"$MYSQL_ROOT_PASSWORD" || mysql_upgrade -uroot -p"$MYSQL_ROOT_PASSWORD"'

bin/moodle-docker-compose up -d --force-recreate webserver
bin/moodle-docker-compose ps
```

## Destructive Clean Rebuild

Use only when intentionally throwing away the local Docker DB and reimporting from a known-good SQL snapshot:

```bash
cd "$MOODLE_DOCKER_DIR"

bin/moodle-docker-compose down -v
bin/moodle-docker-compose up -d db
bin/moodle-docker-wait-for-db

docker cp "$SOURCE_EXPORT/moodle-db-source.sql.gz" "$(bin/moodle-docker-compose ps -q db | /usr/bin/tail -n 1):/tmp/moodle-db-source.sql.gz"

bin/moodle-docker-compose exec -T db sh -lc \
  'gzip -dc /tmp/moodle-db-source.sql.gz | mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE"'

bin/moodle-docker-compose up -d webserver
bin/moodle-docker-compose ps
```

For an initial source export, use the plain SQL file if it has not yet been normalized into `moodle-db-source.sql.gz`:

```bash
cd "$MOODLE_DOCKER_DIR"

docker cp "$SOURCE_EXPORT/<source-sql-file>.sql" "$(bin/moodle-docker-compose ps -q db | /usr/bin/tail -n 1):/tmp/moodle-db-source.sql"

bin/moodle-docker-compose exec -T db sh -lc \
  'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE" < /tmp/moodle-db-source.sql'
```
