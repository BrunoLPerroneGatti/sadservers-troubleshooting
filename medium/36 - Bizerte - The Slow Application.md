## Problem

A Python web application running on port 5000 from the _/opt_ directory is experiencing severe performance issues; every request takes more than 5 seconds to complete.  
The application is supposed to use the _redis-server_ cache service for speed.  
  
Your mission is to diagnose the performance bottleneck and restore the application to its normal, fast response time.  
  
Do not change the Python application file _slow_app.py_.
## Solution

Test the application response time:

```bash
time curl localhost:5000
```

The request takes more than five seconds, and we get `Data from SLOW database!`, confirming the performance issue. 

Inspect the application to understand how it uses Redis:

```bash
cat /opt/slow_app.py
```

The application attempts to connect to Redis using the `REDIS_HOST` environment variable, defaulting to `127.0.0.1`. If the Redis connection fails, it sleeps for five seconds and returns `Data from SLOW database!`.

Check the Redis service status:

```bash
sudo systemctl status redis-server --no-pager
```

Redis is running and listening on `127.0.0.1:6379`, so the problem is likely in the application's configuration.

Inspect the application service configuration:

```bash
sudo systemctl cat slow-app
```

The service configuration contains:

```ini
[Unit]
Description=Slow Flask Application
After=network.target redis-server.service

[Service]
Environment="REDIS_HOST=127.0.0.2"
ExecStart=/usr/bin/python3 /opt/slow_app.py
Restart=always
User=nobody
Group=nogroup

[Install]
WantedBy=multi-user.target
```

The `REDIS_HOST` environment variable is incorrectly set to `127.0.0.2`, while Redis is listening on `127.0.0.1`. As a result, the application's Redis ping fails, triggering the five seconds delay.

Since `/opt/slow_app.py` must not be modified, fix the environment variable in the systemd service configuration. 
Change:`Environment="REDIS_HOST=127.0.0.2"` to: `Environment="REDIS_HOST=127.0.0.1"`

Reload systemd to apply the updated service configuration and restart the application:

```bash
sudo systemctl daemon-reload
sudo systemctl restart slow-app
```

Verify the solution:

```bash
curl localhost:5000
# Expected output:
Data from FAST cache!
```