# Moodle Docker Setup

Starter setup notes for a Moodle 3.11.2 local Docker dev environment.

## Paths

Replace `<active-dev-root>` with the instance being prepared:

```text
/Users/yanoverfieldshaw/Projects/Moodle/DEV(MOODLE_311)
```

The expected local Docker working directories are:

```text
docker-work/additional-plugins/plugins-<version-number>
docker-work/moodle/moodle-<version-number>
docker-work/moodle-docker
docker-work/moodledata/moodledata-<version-number>
docker-work/source-files/source-<version-number>
```
Only moodle and moodle-docker are created by the git install process below - the other directories are for convenience.

Create or populate those directories during the setup stage. Keep `source-files` and `additional-plugins` read-only.

## Get Moodle Docker

Clone the Moodle Docker tooling into the local Docker work area:

```bash
cd "<active-dev-root>/docker-work"

git clone https://github.com/moodlehq/moodle-docker.git moodle-docker
```

If the checkout already exists, update it instead:

```bash
cd "moodle-docker"

git pull --ff-only
```

## Create Moodle Docker `.env`

Create the Moodle Docker environment file at `moodle-docker/.env`.

Use instance-specific ports and project names so multiple rehearsal containers can run side by side. Keep this as plain `.env` syntax, not `export` commands:

```txt
export DEV_ROOT=/Users/yanoverfieldshaw/Projects/Moodle/DEV
export SCRIPTS_ROOT=/Users/yanoverfieldshaw/Projects/Moodle/DEV/docker-work/moodle_scripts
export SOURCE_FILES=/Users/yanoverfieldshaw/Projects/Moodle/DEV/docker-work/source-files/51
export SOURCE_DB=/Users/yanoverfieldshaw/Projects/Moodle/DEV/docker-work/source-files/source-51/moodle-db-source.sql.gz
export SOURCE_MOODLEDATA=/Users/yanoverfieldshaw/Projects/Moodle/DEV/docker-work/source-files/51/moodledata.tar
export MOODLE_DOCKER_MOODLEDATA=/Users/yanoverfieldshaw/Projects/Moodle/DEV/docker-work/moodledata/moodledata-51
export MOODLE_DOCKER_WWWROOT=/Users/yanoverfieldshaw/Projects/Moodle/DEV/docker-work/moodle/moodle-51
export MOODLE_DOCKER_DIR=/Users/yanoverfieldshaw/Projects/Moodle/DEV/docker-work/moodle-docker
export MOODLE_DOCKER_DB=mariadb
export MOODLE_DOCKER_DB_VERSION=11.8
export MOODLE_DOCKER_PHP_VERSION=8.4
export MOODLE_DOCKER_WEB_PORT=8051
export COMPOSE_PROJECT_NAME=local-moodle-51
export MOODLE_DOCKER_DBNAME=moodle
export MYSQL_ROOT_PASSWORD=m@0dl3ing
```

After creating `.env`, load it in the current shell before running Moodle Docker commands:

```bash
cd "<active-dev-root>/docker-work/moodle-docker"
source ../env/.env-<version-number>
```

## Get Moodle 5.1.7

Clone Moodle into the local Moodle work area and check out the initial baseline, Moodle 3.11.2:

```bash
cd "<active-upgrade-root>/docker-work"

git clone https://github.com/moodle/moodle.git moodle
cd "<active-upgrade-root>/docker-work/moodle"
git checkout v5.1.7
```

## local moodle-docker setup

Copy the config.php from moodle-docke into the moodle directory:

```bash
cd "<active-upgrade-root>/docker-work/moodle/moodle-51"
cp ../../moodle-docker/config.docker-template.php config.php
```

Make sure the $CFG->prefix line in config.php matches the table names in the exported Moodle db - the default is 'm_' but Bilkent uses = 'mdl_'

```bash
$CFG->prefix    = 'mdl_';
```

Create a local.yml to specify the local configuration. The following points moodle-docker to a moodledata directory in docker-work, as exported data is usually larger than moodle-docker 1GB webserver capacity.

```yaml
services:
  webserver:
    environment:
      APACHE_DOCUMENT_ROOT: /var/www/html/public
    volumes:
      - "${MOODLE_DOCKER_MOODLEDATA}:/var/www/moodledata"
  db:
    volumes:
      - moodle-db-data:/var/lib/mysql

volumes:
  moodle-db-data:
```

## install exported Moodle code

unzip moodle source code and identify custom and additional plugins. Copy these directories to the moodle directory in the appropriate place in the moodle code tree, e.g. local/bilkent or mod/board etc. For convenience and clarity, you may also copy the directories to the docker-work/additional-plugins directory. The following rsync command achieves this.

```bash
cd additional-plugins/plugins-51

rsync -av --relative \
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
  public/theme/moove \
  ../../moodle/moodle-51/
```
Key detail: do not add trailing slashes to the plugin paths.
Use public/mod/board, not public/mod/board/, so rsync preserves the full target path.
The moodle directory needs a trailing slash

## install exported Moodle DB

initialize the Moodle db
```bash
bin/moodle-docker-compose up -d db
bin/moodle-docker-wait-for-db
```

For an initial source export, use the plain SQL file:

```bash
cd "$MOODLE_DOCKER_DIR"

docker cp "$SOURCE_FILES/moodle-db-source.sql" "$(bin/moodle-docker-compose ps -q db | /usr/bin/tail -n 1):/tmp/moodle-db-source.sql"

bin/moodle-docker-compose exec -T db sh -lc \
  'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE" < /tmp/moodle-db-source.sql'
```

If the db was compressed / normalized into `moodle-db-source.sql.gz`:

```bash
docker cp "$SOURCE_FILES/moodle-db-source.sql.gz" "$(bin/moodle-docker-compose ps -q db | /usr/bin/tail -n 1):/tmp/moodle-db-source.sql.gz"

bin/moodle-docker-compose exec -T db sh -lc \
  'gzip -dc /tmp/moodle-db-source.sql.gz | mariadb -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE"'

```
Start the webserver
```bash
bin/moodle-docker-compose up -d webserver
bin/moodle-docker-compose ps
```

## Next steps

To reset / redo any stage of the process, see `moodle-docker-frequent-commands.md`

After import and configuration, run the health block from `admin-cheatsheet-moodle-docker.md`.