## Problem

A web application relies on the PostgreSQL database on this server. After a client-authentication change, that connection fails.  
  
The application connects over TCP to 127.0.0.1, database _app1_, user _app1user_, password _app1user_. The role and password are already set. Do not replace the database or the account; restore this connection. It must still work after a reboot.

## Solution

```bash
# Test the connection and confirm the authentication error
PGPASSWORD=app1user psql -h 127.0.0.1 -d app1 -U app1user -c '\q'

# Find the path to the active PostgreSQL authentication configuration file
sudo -u postgres psql -c "SHOW hba_file;"
```

Inspect ``pg_hba.conf`` and identify the rules rejecting connections

```
# Client authentication for this cluster.
# Loaded from the data directory (see hba_file). First matching rule wins.
# TYPE  DATABASE        USER            ADDRESS                 METHOD

local   all             postgres                                peer
local   all             all                                     peer

host    all             all             0.0.0.0/0               reject
host    all             all             ::/0                    reject

host    app1            app1user        127.0.0.1/32            trust
host    app1            app1user        ::1/128                 trust
```

Edit pg_hba.conf and move the application-specific rules above the general reject rules. After this, the cluster must be reloaded.

Get PostgreSQL cluster version and name with the command 
```bash
pg_lsclusters
```

Reload the PostgreSQL 17 main cluster to apply the changes
  ```bash
  sudo pg_ctlcluster 17 main reload
  ```
