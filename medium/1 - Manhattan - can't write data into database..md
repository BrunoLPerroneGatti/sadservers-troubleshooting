## Problem

Your objective is to be able to insert a row in an existing Postgres database. The issue is not specific to Postgres and you don't need to know details about it (although it may help).  
  
Helpful Postgres information: it's a service that listens to a port (:5432) and writes to disk in a data directory, the location of which is defined in the _data_directory_ parameter of the configuration file _/etc/postgresql/14/main/postgresql.conf_. In our case Postgres is managed by _systemd_ as a unit with name _postgresql_.

## Solution

Attempt to insert a row into the existing PostgreSQL database to check whether it is working:

```bash
sudo -u postgres psql -c "insert into persons(name) values ('jane smith');" -d dt
```

The connection fails because the PostgreSQL server is not running and the socket `/var/run/postgresql/.s.PGSQL.5432` does not exist.

Check the PostgreSQL service:

```bash
sudo systemctl status postgresql
```

The service shows `active (exited)`. This is the main PostgreSQL wrapper service, which can appear active even when the actual database server is not running. It does not confirm that a PostgreSQL cluster is online. 

Check the actual database cluster:

```bash
sudo pg_lsclusters
```

The output shows that cluster `14/main` is down and uses `/opt/pgdata/main` as its data directory.

Check filesystem usage:

```bash
df -h
```

The output shows that `/opt/pgdata` is mounted on `/dev/nvme0n1`, which is completely full:

```bash
/dev/nvme0n1  8.0G  8.0G  28K  100%  /opt/pgdata
```

Inspect the PostgreSQL cluster logs to confirm the cause:

```bash
sudo journalctl -u postgresql@14-main --no-pager -n 50
```

The logs contain the following error:

```bash
FATAL: could not create lock file "postmaster.pid": No space left on device
```

PostgreSQL cannot start because the disk containing its data directory has no available space.

Identify the largest files and directories on the volume:

```bash
sudo du -ah /opt/pgdata | sort -h | tail -30
```

The output reveals two large backup files outside the PostgreSQL data directory:

```bash
/opt/pgdata/file1.bk   7.0G
/opt/pgdata/file2.bk   923M
```

These backup files consume nearly all available disk space. The actual PostgreSQL data directory, `/opt/pgdata/main`, occupies approximately 50 MB.

Remove the unnecessary backup files:

```bash
rm /opt/pgdata/file*.bk
```

Start PostgreSQL:

```bash
sudo systemctl start postgresql
```

Rerun the original database insertion test:

```bash
sudo -u postgres psql -c "insert into persons(name) values ('jane smith');" -d dt

# Expected output:
INSERT 0 1
```

