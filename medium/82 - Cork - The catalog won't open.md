## Problem

After last week's host reboot the product catalog Postgres database will not stay up. Inserts into the _catalog_ database fail. Ops moved application data off the root disk onto a dedicated volume some time ago; root itself still looks healthy.  
  
You do not need deep Postgres expertise. The server listens on :5432 when healthy. The data directory is set in _/etc/postgresql/*/main/_ (see _data_directory_). _systemd_ manages the unit named _postgresql_.

**Test:** As _admin_: sudo -u postgres psql -d catalog -c "INSERT INTO products(name) VALUES ('jane smith');"  
  
Should return: INSERT 0 1
## Solution

Attempt the required test insert:

```bash
sudo -u postgres psql -d catalog -c "INSERT INTO products(name) VALUES ('jane smith');"
```

The command failed because PostgreSQL was not running and the local socket `/var/run/postgresql/.s.PGSQL.5432` did not exist.

Check the service:

```bash
sudo systemctl status postgresql
sudo pg_lsclusters
```

The `postgresql.service` meta-service showed `active (exited)`, but the actual PostgreSQL 17 cluster was down.

```bash
Ver Cluster Port Status Owner    Data directory
17  main    5432 down   postgres /data/pg/main
```

Inspect the startup logs:

```bash
sudo journalctl -u postgresql@17-main --no-pager -n 50
```

The logs revealed the cause:

```bash
FATAL: could not write lock file "postmaster.pid": No space left on device
```

PostgreSQL could not create its PID lock file because the filesystem containing its data directory was full.

Check filesystem usage:

```bash
df -h
df -h /data
```

The root filesystem was only 70% full, but `/data` was completely full:

```bash
Filesystem      Size  Used  Avail  Use%
/dev/nvme1n1    974M  910M     0  100%
```

PostgreSQL's data directory was `/data/pg/main`, investigate disk usage under `/data`:

```bash
sudo du -xhd1 /data 2>/dev/null | sort -h
sudo du -xhd1 /data/pg 2>/dev/null | sort -h
```

The results showed:
- `/data/pg`: approximately 47 MB.    
- `/data/docker`: approximately 864 MB.    

Docker was consuming almost all the available space.

Inspect Docker's disk usage:

```bash
sudo docker system df
sudo docker ps -a --size
sudo docker images
sudo du -ah /data/docker/containers 2>/dev/null | sort -h | tail -30
```

Docker's images and container writable layers were small. Almost all the space was used by a single container log belonging to `metrics-agent`.

Inspect the log path and logging configuration:

```bash
sudo docker inspect --format '{{.LogPath}}' metrics-agent
sudo docker inspect --format '{{json .HostConfig.LogConfig}}' metrics-agent
```

The container used the `json-file` logging driver without rotation limits, allowing the `metrics-agent` container's JSON log to grow without a size limit, filling the dedicated `/data` volume and preventing PostgreSQL from starting. 

The log file was approximately 864 MB.

Truncate the oversized log without removing the container:

```bash
sudo truncate -s 0 /data/docker/containers/c434ca1c3a401cdd8b724acd83b6fc688af35e39e96816a9d1309c4707e9c876/c434ca1c3a401cdd8b724acd83b6fc688af35e39e96816a9d1309c4707e9c876-json.log
```

This permanently discarded the existing log contents.

Restart PostgreSQL

```bash
sudo systemctl start postgresql@17-main
sudo pg_lsclusters
```

The cluster changed to `online`:

```bash
Ver Cluster Port Status Owner    Data directory
17  main    5432 online postgres /data/pg/main
```

Run the required test insert:

```bash
sudo -u postgres psql -d catalog -c "INSERT INTO products(name) VALUES ('jane smith');"
# Expected output:
INSERT 0 1
```

