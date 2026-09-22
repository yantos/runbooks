# Moodle 5.1 local/bilkent plugin install

This runbook documents how to install the bilkent plugin from the Moodle/DEV/plugins folder

Use rsync:

local docker container install:

```bash
  rsync -av --dry-run --delete \
  --exclude='.DS_Store' \
  --exclude='.git/' \
  --exclude='.gitignore' \
  --exclude='.notes' \
  --exclude='scripts/logs/' \
  . /Users/yanoverfieldshaw/Projects/Moodle/DEV/MOODLE_51/docker-work/moodle/public/local/bilkent/
```

remote test.moodle install:

```bash
rsync -av --dry-run --delete \
  --exclude='.DS_Store' \
  --exclude='.git/' \
  --exclude='.gitignore' \
  --exclude='.notes' \
  --exclude='scripts/logs/' \
  . yan@test.moodle.bilkent.edu.tr:/var/www/moodle/public/local/bilkent/
```