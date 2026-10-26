## Problem

There is a Traefik load balancer that must be up and running. The server and the backend services are managed by Docker Compose. Running curl -s app.sadserver | head -n1 must return the host ID of one of the backend servers, running the command again must return a new host ID. The server seems to be working some times, some others fails or just times out.  
  
The round-robin configuration should make the webserver iterate through the back-end servers.

## Solution

Test the application by making repeated requests:

```bash
curl -s app.sadserver | head -n1
```

Some requests return `Hostname: <container-id>`, while others return `Bad Gateway` or hang. This indicates that Traefik is running but some of its backend connections are failing.

List the running and stopped containers:

```bash
docker ps -a
```

Traefik and all four backend containers (`app-app01-1`, `app-app02-1`, `app-app03-1`, and `app-app04-1`) are running.

Inspect the Traefik logs:

```bash
docker logs --tail 50 app-traefik-1
```

The logs reveal a connection error:

```bash
502 Bad Gateway error="dial tcp 172.19.0.4:81: connect: connection refused"
```

Traefik is attempting to connect to port `81` on one of the backends, but the `traefik/whoami` containers listen on port `80`.

Find the Docker Compose configuration:

```bash
docker compose ls
cat /home/admin/app/docker-compose.yml
```

Inspecting the Compose file reveals two configuration errors:

1. **Incorrect backend port:** `app02` has the label `traefik.http.services.app.loadbalancer.server.port` set to `"81"` instead of `"80"`. Traefik therefore sends requests to a port where the backend is not listening, causing `502 Bad Gateway` responses.
2. **Incorrect Docker network:** `app04` is connected to the `internal` network instead of the `web` network. Traefik is connected to `web`, so it cannot reliably reach `app04` through the expected network.

Edit the Compose file.
For `app02`, change the backend port label to:

```yaml
traefik.http.services.app.loadbalancer.server.port: "80"
```

For `app04`, change its network configuration to:

```yaml
networks:
  - web
```

Recreate the containers to apply the changes:

```bash
docker compose up -d --force-recreate
```


Test the application againg by making repeated requests.
The output should contain successful `Hostname: <container-id>` responses from all four backends.
