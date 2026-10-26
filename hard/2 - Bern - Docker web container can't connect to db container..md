## Problem

There are two Docker containers running, a web application (Wordpress or WP) and a database (MariaDB) as back-end, but if we look at the web page, we see that it cannot connect to the database. curl -s localhost:80 |tail -4 returns:  
  
`<body id="error-page"> <div class="wp-die-message"><h1>Error establishing a database connection</h1></div></body> </html>`

This is not a Wordpress code issue (the image is :latest with some network utilities added). What you need to know is that WP uses "WORDPRESS_DB_" environment variables to create the MySQL connection string. See the ./html/wp-config.php WP config file for example (from /home/admin).

**Test:** sudo docker exec wordpress mysqladmin -h mysql -u root -ppassword ping . The wordpress container is able to connect to the database in the mariadb container and returns mysqld is alive.
## Solution

Test the required database connection using the hostname `mysql`:

```bash
sudo docker exec wordpress mysqladmin -h mysql -u root -ppassword ping
```

The command failed with:

```bash
mysqladmin: connect to server at 'mysql' failed
error: 'Unknown MySQL server host 'mysql' (-2)'
```

This indicated that the hostname `mysql` could not be resolved from the WordPress container.

Inspect the running containers:

```bash
sudo docker ps -a
```

The output showed two running containers:
- `wordpress`, publishing port 80.    
- `mariadb`, publishing port 3306.    

Inspect the WordPress database environment variables:

```bash
sudo docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' wordpress | grep '^WORDPRESS_DB_'
```

The output was:

```bash
WORDPRESS_DB_PASSWORD=password
WORDPRESS_DB_USER=root
```

`WORDPRESS_DB_HOST` and `WORDPRESS_DB_NAME` were not explicitly defined.

Inspect the WordPress configuration file:

```bash
grep 'WORDPRESS_DB_' ./html/wp-config.php
```

The relevant configuration was:

```bash
define( 'DB_NAME', getenv_docker('WORDPRESS_DB_NAME', 'wordpress') );
define( 'DB_USER', getenv_docker('WORDPRESS_DB_USER', 'example username') );
define( 'DB_PASSWORD', getenv_docker('WORDPRESS_DB_PASSWORD', 'example password') );
define( 'DB_HOST', getenv_docker('WORDPRESS_DB_HOST', 'mysql') );
```

The database name defaulted to `wordpress`, and the database hostname defaulted to `mysql`. The username and password were already configured through environment variables.

Inspect the network attachments of both containers:

```bash
sudo docker inspect -f '{{json .NetworkSettings.Networks}}' wordpress
sudo docker inspect -f '{{json .NetworkSettings.Networks}}' mariadb
```

Both containers were attached only to Docker's default `bridge` network. Their IP addresses were `172.17.0.3` for WordPress and `172.17.0.2` for MariaDB.

Test the database connection using MariaDB's IP address instead of its hostname:

```bash
sudo docker exec wordpress mysqladmin -h 172.17.0.2 -u root -ppassword ping
```

The command returned:

```
mysqld is alive
```

This confirmed that the database was reachable and the credentials were valid. The problem was hostname resolution, `mysql` did not resolve to the MariaDB container.

Check whether the hostnames resolve inside WordPress:

```bash
sudo docker exec wordpress getent hosts mariadb
sudo docker exec wordpress getent hosts mysql
```

Neither command returned an address. The containers were on the default bridge network, which does not provide the same automatic container name and network alias DNS resolution as a user defined bridge network.

Create a user defined bridge network:

```bash
sudo docker network create wp-network
```

Connect MariaDB to the new network and assign it the alias `mysql`:

```bash
sudo docker network connect --alias mysql wp-network mariadb
```

Connect WordPress to the same network:

```bash
sudo docker network connect wp-network wordpress
```

Verify hostname resolution:

```bash
sudo docker exec wordpress getent hosts mysql
```

The command returned:

```
172.18.0.2      mysql
```

The hostname now resolved to the MariaDB container's IP address.

Test the required database connection again:

```bash
sudo docker exec wordpress mysqladmin -h mysql -u root -ppassword ping
```

The command returned:

```
mysqld is alive
```
