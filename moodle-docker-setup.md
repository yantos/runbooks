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

## Shell Setup

After the Moodle Docker checkout and `.env` exist:

```bash
cd "<active-upgrade-root>/docker-work/moodle-docker"
set -a
source .env
set +a
```

Repair the shell if normal commands such as `ls`, `grep`, `cp`, or `dirname` disappear:

```bash
export PATH="/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"; hash -r
```

Confirm important paths:

```bash
printf 'UPGRADE_ROOT=%s\nMOODLE_DOCKER_DIR=%s\nMOODLE_DOCKER_WWWROOT=%s\nSOURCE_EXPORT=%s\nSOURCE_HTML=%s\nMOODLE_DOCKER_MOODLEDATA=%s\n' \
  "$UPGRADE_ROOT" "$MOODLE_DOCKER_DIR" "$MOODLE_DOCKER_WWWROOT" "$SOURCE_EXPORT" "$SOURCE_HTML" "$MOODLE_DOCKER_MOODLEDATA"
```

## Finder Metadata

Quietly remove macOS Finder metadata from the Moodle code tree before upgrades and snapshots:

```bash

```

Finder can recreate `.DS_Store` files whenever the Moodle tree is open in a Finder window. The simplest prevention is to keep these upgrade folders out of Finder while running the rehearsal and use Terminal/Codex for browsing them.

## First Baseline

The first stage should establish Moodle 3.11.2 in Docker before moving to 3.11.10. After import and configuration, run the health block from `admin-cheatsheet-moodle-docker.md`.
