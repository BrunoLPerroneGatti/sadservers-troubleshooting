## Problem

There is a web server (Caddy) listening on HTTP port 80, but `curl http://127.0.0.1` does not work. Find out what is wrong and make the necessary fixes, and the web server will give you a URL.

Note: As a constraint, the file `/home/admin/db_connector.py` must not be modified for the solution to be considered correct.

The web server must respond to the IP address `127.0.0.1`, not just to `localhost`.

## Solution

Test the web server:

```bash
curl http://127.0.0.1
```

The request hangs, so inspect the listening ports and firewall rules:

```bash
sudo ss -ltnp | grep ':80'
sudo iptables -L
```

Caddy is listening on port `80`, but the `INPUT` chain contains a rule dropping incoming TCP traffic on HTTP port 80.

Remove the blocking rule:

```bash
sudo iptables -D INPUT 1
```

Retry the request:

```bash
curl http://127.0.0.1
```

The request now reaches the backend, but returns an error indicating that the connection to `127.0.0.1:5433` is refused.

Inspect the Caddy configuration:

```bash
# Get the configuration file
systemctl cat caddy

cat /etc/caddy/Caddyfile
```

Caddy is configured to reverse-proxy requests to `localhost:5050`. Investigate which service is listening on that port:

```bash
sudo ss -ltnp | grep ':5050'
```

The output shows that a Python process is listening on port `5050`. Identify the process and inspect the running Python processes:

```bash
ps auxf | grep python3
```

The output reveals that the process is running `/home/admin/db_connector.py`.

Inspect the script to understand how it handles requests and connects to the database:

```bash
cat /home/admin/db_connector.py
```

The script accepts HTTP requests on port `5050` and connects to PostgreSQL using connection parameters loaded from the `.env` file.

Check the PostgreSQL service status:

```bash
systemctl status postgresql
```

PostgreSQL is inactive. Start the service:

```bash
sudo systemctl start postgresql
```

Verify which port PostgreSQL is listening on:

```bash
sudo ss -ltnp
```

The output shows PostgreSQL listening on `127.0.0.1:5432`, while the backend error reports a connection attempt to port `5433`. The database port configured for the backend therefore needs to be corrected.

Inspect the database connection settings:

```bash
cat /home/admin/.env
```

The environment file contains:

```
DBNAME="nerdearla"
DBUSER="usuario"
DBPASSWORD="password"
DBHOST="127.0.0.1"
DBPORT="5433"
```

The configured port does not match the actual PostgreSQL listening port. Since `/home/admin/db_connector.py` must not be modified, fix the connection settings in `.env`.

Edit the file and change `DBPORT` from `5433` to `5432` and restart the backend service so it reloads the environment variables:

```bash
sudo systemctl restart db_connector
```

Verify the solution:

```bash
curl http://127.0.0.1
# Expected output:
https://sadservers.com/nerdearla2024
```
