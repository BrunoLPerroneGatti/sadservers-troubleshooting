## Problem

Every deployment through the Gitea Actions pipeline reports full success: checkout, build, deploy, service reload, and health check all pass with green checkmarks. Despite this, users keep seeing the **previous** version of the app after every deploy.  
  
`~/webapp` is a git clone of the Gitea repo. Edit a file there, commit, and git push to trigger the Actions pipeline. Then figure out why the running webapp service never actually serves the version it just deployed — and fix it so a successful pipeline run always results in that version being served. For example:  

cd ~/webapp && echo "v2" > index.html && git commit -am "bump" && git push

## Solution

Inspect the deployed application and compare it with the content served over HTTP:

```bash
cat /srv/app/current/index.html
curl -s http://localhost:80/
```

The deployed file contains `Version: abc1234`, but the HTTP response contains `Version: initial`. This confirms that the deployment pipeline reports success even though the web server continues serving an older version.

Inspect the Gitea Actions workflow:

```bash
cat ~/webapp/.gitea/workflows/deploy.yml
```

The workflow builds an HTML file, copies it into a timestamped release directory under `/srv/releases/`, updates `/srv/app/current` to point to the new release, and reloads the service:

```yaml
      - name: Deploy
        run: |
          TS=$(date +%s)
          RELEASE_DIR="/srv/releases/${TS}"
          sudo mkdir -p "$RELEASE_DIR"
          sudo cp "$(pwd)/index.html" "$RELEASE_DIR/"
          sudo ln -sfn "$RELEASE_DIR" /srv/app/current

      - name: Reload service
        run: sudo systemctl reload webapp.service
```

Inspect the service configuration:

```bash
sudo systemctl cat webapp
```

The service starts Nginx after creating a bind mount:

```ini
ExecStart=/bin/sh -c 'mount --bind "$(readlink -f /srv/app/current)" /srv/app/live && /usr/sbin/nginx'
ExecReload=/bin/kill -HUP $MAINPID
```

The root cause is that the deployment updates the `/srv/app/current` symlink, but the existing bind mount at `/srv/app/live` continues referencing the previous release directory. Reloading Nginx only sends it a reload signal, it does not recreate the bind mount.

Fix the workflow by changing the service reload to a restart:

```ini
      - name: Reload service
        run: sudo systemctl restart webapp.service
```

A restart reruns the service's startup command, recreating the bind mount from the updated `/srv/app/current` symlink.

Commit and push the workflow change to trigger a new deployment:

```bash
cd ~/webapp
git add .gitea/workflows/deploy.yml
git commit -m "Restart webapp after deployment"
git push origin main
```

Compare the deployed file with the HTTP response:

```bash
cat /srv/app/current/index.html
curl -s http://localhost:80/
```

Both now must return the same content.
