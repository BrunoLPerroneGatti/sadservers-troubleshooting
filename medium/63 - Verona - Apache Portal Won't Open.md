## Problem

An internal Apache portal was migrated to this host. The document root is _/var/www/portal_.  
  
The site root at http://localhost/ does not serve the expected homepage, and a legacy bookmark at /reports no longer reaches the status page (direct access to /status/ works).  
  
Find and fix what keeps the portal root and the legacy redirect from working. Adding missing content is allowed.

## Solution

Test the portal homepage and the legacy bookmark:

```bash
curl -i http://localhost/
curl -L http://localhost/reports
curl -L http://localhost/status/
```

The homepage returns `403 Forbidden`, and `/reports` returns `404 Not Found`. However, `/status/` works and returns `SadServers - Status OK`, confirming that the status page itself is functioning.

Inspect the active Apache virtual host configuration:

```bash
sudo apache2ctl -S
sudo cat /etc/apache2/sites-enabled/portal.conf
```

The active virtual host uses `/var/www/portal` as its document root. Its directory configuration includes:

```conf
<Directory /var/www/portal>
    Options -Indexes +FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

The `AllowOverride None` directive prevents Apache from processing applicable directives in `.htaccess` files. Additionally, `Options -Indexes` disables directory listings, so a request to a directory without a default index file returns `403 Forbidden`.

Inspect the portal directory and its `.htaccess` file:

```bash
sudo ls -la /var/www/portal
cat /var/www/portal/.htaccess
```

The directory contains no homepage file, such as `index.html`. The `.htaccess` file contains the intended legacy redirect:

```
RedirectMatch ^/reports/?$ /status/
```

However, Apache ignores this redirect because `AllowOverride None` disables `.htaccess` processing.

Fix the Apache configuration by editing the virtual host:

```bash
sudo nano /etc/apache2/sites-enabled/portal.conf
```

Change `AllowOverride None` to `AllowOverride All`.
This enables Apache to process the existing redirect rule in `.htaccess`, allowing requests to `/reports` to redirect to `/status/`.

Create the missing homepage:

```bash
echo "SadServers - Portal Ready" > /var/www/portal/index.html
```

Validate the Apache configuration and reload the service:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

The configuration test should report `Syntax OK`. 

Verify the homepage:

```bash
curl -i http://localhost/
```

The response should be `HTTP/1.1 200 OK` and contain the new portal homepage.

Verify the legacy redirect:

```bash
curl -I http://localhost/reports
```

The response should be `302 Found`, with a `Location` header pointing to `/status/`.

Verify the challenge's expected output:

```bash
curl -L http://localhost/reports
# Expected output:
SadServers - Status OK
```
