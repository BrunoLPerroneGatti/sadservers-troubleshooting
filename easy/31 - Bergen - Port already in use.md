## Problem

There's an application at _/home/admin/standalone_ that needs to run successfully but currently it fails.  
  
Fix the environment so the binary can run without errors, without changing the binary itself, and without breaking the web app served on port :80.
## Solution

Check that the web app on port 80 is responding correctly and run the binary to identify the error:

```bash
curl http://localhost:80
/home/admin/standalone
```

Output:
```bash
hello SadServers
listen tcp4 0.0.0.0:8000: bind: address already in use
```

The binary cannot start because port 8000 is already occupied.

Identify the process using port 8000:
```bash
sudo ss -tlnp | grep 8000
```

The output shows that Nginx is listening on port 80 and Python is listening on port 8000, with PID 923.

Identify the service associated with the process:
```bash
systemctl status 923
```

The output identifies the service as `django.service`.

Inspect the Django service:
```bash
sudo systemctl cat django.service
```

```ini
# /etc/systemd/system/django.service
[Unit]
Description=Django Hello SadServers (dev runserver)
After=network.target

[Service]
Type=simple
User=admin
Group=admin
WorkingDirectory=/home/admin/django_app
ExecStart=/usr/bin/python3 manage.py runserver 0.0.0.0:8000
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Django is configured to listen on port 8000, which conflicts with `/home/admin/standalone`.


Confirm that Nginx is running:

```
systemctl status nginx
```

The main Nginx configuration, located on ``sudo cat /etc/nginx/nginx.conf``, contains: 

```
include /etc/nginx/sites-enabled/*;
```

This tells that Nginx loads configuration files from `/etc/nginx/sites-enabled/`. 

Inspect the `bergen` configuration:
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```


Nginx listens on port 80 and forwards incoming requests to Django on port 8000.

Move Django to a different port and update Nginx to use the new port. Otherwise, changing Django's port would break the existing web application.


Changes:
```bash
# /etc/systemd/system/django.service
ExecStart=/usr/bin/python3 manage.py runserver 0.0.0.0:8001

# /etc/nginx/sites-enabled
proxy_pass http://127.0.0.1:8001;
```

Reload systemd and restart the services:

```
sudo systemctl daemon-reload
sudo systemctl restart django
sudo nginx -t && sudo systemctl reload nginx
```