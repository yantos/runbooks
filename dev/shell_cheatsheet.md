# Frequent bash and zsh Commands

bash and zsh commands we often need and don't remember :)

## Shell Setup

After the Moodle Docker checkout and `.env` exist:

```bash
cd "cd /Users/yanoverfieldshaw/Projects/Moodle/UPGRADES/<active-upgrade-root>/docker-work/moodle-docker"

source ../env/.env-<version-number>
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

Repair the shell if normal commands such as `ls`, `grep`, `cp`, or `dirname` disappear:

```bash
export PATH="/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"; hash -r
```

## Strip macOS artifacts

Finder can recreate `.DS_Store` files whenever the Moodle tree is open in a Finder window. The simplest prevention is to keep these upgrade folders out of Finder while running the rehearsal and use Terminal/Codex for browsing them.

```bash
/usr/bin/find . \( -name '.DS_Store' -o -name '._*' \) -type f -print -delete
```