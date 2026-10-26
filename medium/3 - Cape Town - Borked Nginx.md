## Problem

There's an Nginx web server installed and managed by systemd. Running curl -I 127.0.0.1:80 returns curl: (7) Failed to connect to localhost port 80: Connection refused , fix it so when you curl you get the default Nginx page.

## Solution

Check the Nginx service status:

```bash
sudo systemctl status nginx
```

The service failed to start because of a syntax error in `/etc/nginx/sites-enabled/default`.

```
nginx: [emerg] unexpected ";" in /etc/nginx/sites-enabled/default:1
```

The first line of he configuration file contained an unexpected semicolon. Delete it and validate the configuration:

```bash
sudo nginx -t
```

Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

Although Nginx started successfully, if we try to connect with `curl -I 127.0.0.1:80` the server returns `500 Internal Server Error`.

Inspect the Nginx error log:

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

The relevant errors were:

```
open() "/var/www/html/index.nginx-debian.html" failed (24: Too many open files)
accept4() failed (24: Too many open files)
```

Inspect the systemd service configuration:
```bash
sudo systemctl cat nginx
```

The service contained an incorrect file descriptor limit:

```ini
LimitNOFILE=10
```

This limit was too low for Nginx to operate correctly. Remove it and reload systemd along with Nginx:

```bash
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

Run the required test:

```bash
curl -Is 127.0.0.1:80 | head -1
# Expected output:
HTTP/1.1 200 OK
```