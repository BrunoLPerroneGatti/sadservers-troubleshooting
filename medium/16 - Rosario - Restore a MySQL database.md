## Problem

A developer created a database named 'main' but now some data is missing in the database. You need to restore the database using the the dump "/home/admin/backup.sql".  
The issue is that the developer forgot the root password for the MariaDB server.  
If you encounter an issue while restoring the database, fix it.

## Solution

Diagnose MariaDB

```bash
sudo systemctl status mariadb
```

MariaDB is running, but logs show `Access denied for user 'root'@'localhost'`. The root password is unknown, so normal database access fails.

Enable recovery mode:

```
sudo systemctl stop mariadb
sudo systemctl set-environment MYSQLD_OPTS="--skip-grant-tables --skip-networking"
sudo systemctl start mariadb
```

`MYSQLD_OPTS` is an environment variable used to pass startup options to the MariaDB server through systemd. 
`--skip-grant-tables` bypasses authentication checks, allowing root access without a password.
`--skip-networking` disables TCP/IP connections during recovery.

Try to restore the database:

```
sudo mysql -u root main < /home/admin/backup.sql
```

Import fails with `ERROR 1064` at line 7. Inspection reveals unexpected `?` characters at the ends of SQL statements.

Fix the corrupted dump:

```
sed 's/?/;/g' /home/admin/backup.sql > /home/admin/backup2.sql
```

`sed` replaces each `?` with `;`, saving the repaired dump as `backup2.sql` while preserving the original.

Try to restore the database again:

```
sudo mysql -u root main < /home/admin/backup2.sql
```

No output or error indicates the import completed successfully.

Verify the solution:

```
sudo mysql -u root -e "SHOW TABLES FROM main;"
```

The output contains `solution`, confirming the challenge's required table exists.
