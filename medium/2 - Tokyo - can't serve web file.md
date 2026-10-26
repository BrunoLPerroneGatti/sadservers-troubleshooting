## Problem

There's a web server serving a file _/var/www/html/index.html_ with content "hello sadserver" but when we try to check it locally with an HTTP client like curl 127.0.0.1:80, nothing is returned. This scenario is not about the particular web server configuration and you only need to have general knowledge about how web servers work.

## Solution

Attempt to access the web server locally:

```bash
curl 127.0.0.1:80
```

We don't get a response, the connection might be dropped or the server is not responding. 

Check which process is listening on port 80:

```bash
sudo ss -ltnp | grep ':80'
```

The output shows:

```
LISTEN 0 511 *:80 *:* users:(("apache2",pid=865,fd=4),("apache2",pid=852,fd=4),("apache2",pid=851,fd=4))
LISTEN 0 4096 *:8080 *:* users:(("gotty",pid=760,fd=6))
```

Apache is listening on port 80, so the problem is not simply that no web server is listening.

Check the Apache service status:

```bash
sudo systemctl status apache2
```

The output shows that Apache is running normally and listening on the expected port

Check the HTML file and its permissions:

```bash
ls -l /var/www/html/index.html
```

The output is:

```
-rw------- 1 root root 15 Sep 4 15:07 /var/www/html/index.html
```

The permissions `rw-------` mean that only the root user can read or write the file. Apache's worker processes normally run as a different user, so they cannot read it.

Change the permissions to allow the web server to read the file:

```bash
sudo chmod 644 /var/www/html/index.html
```

This grants read permission to everyone while retaining root ownership.

Test the HTTP request again:

```bash
curl 127.0.0.1:80
```

We still don't get a response, so the file permissions were not the only problem worth.

Inspect the firewall rules

```bash
sudo iptables -L
```

The output reveals a rule dropping all incoming TCP traffic to HTTP port 80:

```
Chain INPUT (policy ACCEPT)
target  prot opt source      destination
DROP    tcp  --  anywhere    anywhere    tcp dpt:http
```

Remove the firewall rule that's blocking TCP port 80:

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
```

Verify the solution:

```bash
curl 127.0.0.1:80
hello sadserver
```

