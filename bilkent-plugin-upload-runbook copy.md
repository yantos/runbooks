# Moodle Public Assets Upload Runbook

This runbook documents how to set up and maintain the public static assets
folder used for files such as the Moodle login QR code.


rsync -av --dry-run --delete \
  --exclude='.DS_Store' \
  --exclude='.git/' \
  --exclude='.gitignore' \
  --exclude='.notes' \
  --exclude='scripts/logs/' \
  . yan@test.moodle.bilkent.edu.tr:/var/www/moodle/public/local/bilkent/

## Scope

The public asset URL is:

```text
https://gen.moodle.bilkent.edu.tr/bilkent-moodle-assets/
```

The server-side directory is:

```text
/var/www/moodle/public/bilkent-moodle-assets
```

The deploy user is:

```text
yan
```

The deploy host is:

```text
gen.moodle.bilkent.edu.tr
```

Example QR image filename:

```text
x-moodle-qr-code.png
```

## Security Model

This directory is intentionally public. It should contain only static files that
are safe for anonymous visitors to download, such as PNG images.

Do not put secrets, backups, configuration files, private documents, login
tokens, or generated exports in this directory.

The directory should be:

- owned by the deploy user, so files can be updated intentionally
- readable by Apache
- not writable by the Apache/web-server user
- protected against directory listing
- protected against script execution

Unix file permissions alone are not enough to block PHP execution. Apache/PHP
can execute a readable `.php` file even when the Unix executable bit is not set,
so the Apache virtual host should explicitly deny script files in this directory.

## One-Time Directory Setup

SSH into the server:

```bash
ssh yan@gen.moodle.bilkent.edu.tr
```

Create the directory:

```bash
sudo mkdir -p /var/www/moodle/public/bilkent-moodle-assets
```

Make the deploy user own the directory and any existing contents:

```bash
sudo chown -R yan:yan /var/www/moodle/public/bilkent-moodle-assets
```

Set directory and file permissions:

```bash
sudo find /var/www/moodle/public/bilkent-moodle-assets -type d -exec chmod 755 {} \;
sudo find /var/www/moodle/public/bilkent-moodle-assets -type f -exec chmod 644 {} \;
```

These permissions allow `yan` to update the assets while allowing Apache to
read and serve them.

## Apache Virtual Host Hardening

Because the Apache virtual host config is available, prefer vhost rules instead
of `.htaccess`.

Add this block to the relevant Apache virtual host or included site config:

```apache
<Directory "/var/www/moodle/public/bilkent-moodle-assets">
    Options -Indexes
    AllowOverride None

    <FilesMatch "\.(php|phtml|phar|cgi|pl|sh)$">
        Require all denied
    </FilesMatch>
</Directory>
```

Check the Apache configuration:

```bash
sudo apachectl configtest
```

Reload Apache:

```bash
sudo systemctl reload apache2
```

If the server uses a different Apache service name, use the appropriate service
for that host.

## Routine Deployment With Rsync

For routine deployments, prefer `rsync` from the local assets directory to the
server-side public assets directory.

From the local machine, first change into the local directory whose contents
should be deployed. Then run a dry run:

```bash
rsync -av --dry-run --itemize-changes ./ yan@gen.moodle.bilkent.edu.tr:/var/www/moodle/public/bilkent-moodle-assets/
```

Review the listed changes. If they look correct, run the normal sync:

```bash
rsync -av --progress ./ yan@gen.moodle.bilkent.edu.tr:/var/www/moodle/public/bilkent-moodle-assets/
```

The `./` source is important: it syncs the contents of the current local
directory into `bilkent-moodle-assets`, rather than creating another directory
inside it.

This routine sync adds or updates files on the server, but it does not delete
extra files that already exist remotely.

## Optional True-Mirror Mode

Use `--delete` only when the remote directory should exactly match the current
local directory. This removes remote files that no longer exist locally, so
always dry-run first and review the deletion list carefully:

```bash
rsync -av --delete --dry-run --itemize-changes ./ yan@gen.moodle.bilkent.edu.tr:/var/www/moodle/public/bilkent-moodle-assets/
```

If the dry run is correct, run the mirror sync:

```bash
rsync -av --delete --progress ./ yan@gen.moodle.bilkent.edu.tr:/var/www/moodle/public/bilkent-moodle-assets/
```

## Fallback: Upload A Single File With Scp

Use `scp` as a fallback for a one-off single-file upload, or when `rsync` is not
available.

From the local machine, copy the file to the user's home directory on the
server:

```bash
scp x-moodle-qr-code.png yan@gen.moodle.bilkent.edu.tr:/home/yan/
```

SSH into the server:

```bash
ssh yan@gen.moodle.bilkent.edu.tr
```

Move the file into the public assets directory:

```bash
mv /home/yan/x-moodle-qr-code.png /var/www/moodle/public/bilkent-moodle-assets/
```

Set ownership and permissions on the uploaded file:

```bash
chown yan:yan /var/www/moodle/public/bilkent-moodle-assets/x-moodle-qr-code.png
chmod 644 /var/www/moodle/public/bilkent-moodle-assets/x-moodle-qr-code.png
```

If the move or ownership command requires elevated privileges, use:

```bash
sudo mv /home/yan/x-moodle-qr-code.png /var/www/moodle/public/bilkent-moodle-assets/
sudo chown yan:yan /var/www/moodle/public/bilkent-moodle-assets/x-moodle-qr-code.png
sudo chmod 644 /var/www/moodle/public/bilkent-moodle-assets/x-moodle-qr-code.png
```

## Verify

Check the file on the server:

```bash
ls -l /var/www/moodle/public/bilkent-moodle-assets/x-moodle-qr-code.png
```

Expected permissions should look like:

```text
-rw-r--r-- 1 yan yan ... x-moodle-qr-code.png
```

Check the public URL:

```bash
curl -I https://gen.moodle.bilkent.edu.tr/bilkent-moodle-assets/x-moodle-qr-code.png
```

Expected result:

```text
HTTP/2 200
content-type: image/png
```

Check that directory listing is disabled:

```bash
curl -I https://gen.moodle.bilkent.edu.tr/bilkent-moodle-assets/
```

A `403 Forbidden` response is acceptable and expected.

## Operational Notes

- Keep only public static assets in this directory.
- Do not make the directory writable by the Apache user.
- Do not use `chmod 777`.
- Re-run the `find ... chmod` commands after bulk uploads if needed.
- If a QR code changes, verify that it points to the intended HTTPS Moodle URL.
