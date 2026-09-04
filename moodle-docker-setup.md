# Moodle Docker Setup

Starter setup notes for a Moodle 3.11.2 local Docker rehearsal.

## Paths

Replace `<active-upgrade-root>` with the instance being prepared:

```text
/Users/yanoverfieldshaw/Projects/Moodle/UPGRADES/mainmoodle-upgrade-2026
/Users/yanoverfieldshaw/Projects/Moodle/UPGRADES/prepmoodle-upgrade-2026
```

The expected local Docker working directories are:

```text
docker-work/moodle
docker-work/moodle-docker
```

Create or populate those directories during the setup stage. Keep `source-export`, `additional-plugins`, and `production-import` read-only.

## Get Moodle Docker

Clone the Moodle Docker tooling into the local Docker work area:

```bash
cd "<active-upgrade-root>/docker-work"

git clone https://github.com/moodlehq/moodle-docker.git moodle-docker
```

If the checkout already exists, update it instead:

```bash
cd "<active-upgrade-root>/docker-work/moodle-docker"

git pull --ff-only
```

## Create Moodle Docker `.env`

Create the Moodle Docker environment file at `<active-upgrade-root>/docker-work/moodle-docker/.env`.

Use instance-specific ports and project names so multiple rehearsal containers can run side by side. Keep this as plain `.env` syntax, not `export` commands:

```text
UPGRADE_ROOT=<active-upgrade-root>
MOODLE_DOCKER_DIR=<active-upgrade-root>/docker-work/moodle-docker
MOODLE_DOCKER_WWWROOT=<active-upgrade-root>/docker-work/moodle
MOODLE_DOCKER_MOODLEDATA=<active-upgrade-root>/moodle-work/moodledata

COMPOSE_PROJECT_NAME=<instance-name>-moodle-upgrade-2026
MOODLE_DOCKER_DB=mariadb
MOODLE_DOCKER_DB_VERSION=10.6
MOODLE_DOCKER_DBNAME=moodle
MOODLE_DOCKER_WEB_HOST=localhost
MOODLE_DOCKER_WEB_PORT=<local-web-port>
MOODLE_DOCKER_PHP_VERSION=7.4
MYSQL_ROOT_PASSWORD=m@0dl3ing
```

Suggested instance-specific values:

```text
COMPOSE_PROJECT_NAME=mainmoodle-upgrade-2026
MOODLE_DOCKER_WEB_PORT=8036
```

```text
COMPOSE_PROJECT_NAME=prepmoodle-upgrade-2026
MOODLE_DOCKER_WEB_PORT=8311
```

After creating `.env`, load it in the current shell before running Moodle Docker commands:

```bash
cd "<active-upgrade-root>/docker-work/moodle-docker"
set -a
source .env
set +a
```

## Get Moodle 3.11

Clone Moodle into the local Moodle work area and check out the initial baseline, Moodle 3.11.2:

```bash
cd "<active-upgrade-root>/docker-work"

git clone https://github.com/moodle/moodle.git moodle
cd "<active-upgrade-root>/docker-work/moodle"
git checkout v3.11.2
```

For the later 3.11.10 stage, fetch tags and check out the target tag:

```bash
cd "<active-upgrade-root>/docker-work/moodle"

git fetch --tags
git checkout v3.11.10
```

## First Baseline

The first stage should establish Moodle 3.11.2 in Docker before moving to 3.11.10. After import and configuration, run the health block from `admin-cheatsheet-moodle-docker.md`.
