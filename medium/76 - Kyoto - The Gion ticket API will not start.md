## Problem

The Gion ticket office API should answer on port :80. After last night's change the office never came back.  
  
Manifests are in _/home/admin/app_. There is a short handover note in _/home/admin/HANDOVER.txt_. Wait until the Kubernetes node is Ready after boot.  
  
GET /health must return {"status":"ok","office":"gion-tickets"}. Do not move the API off port 80.

## Solution

Inspect the Kubernetes node and the application pods:

```bash
kubectl get nodes
kubectl get pods -n gion -o wide
kubectl describe pod -n gion -l app=gion-api
```

The node is `Ready`, but the `gion-api` pod is in `CrashLoopBackOff`. The container exits with code `1`, indicating that the application fails during startup.

Inspect the container logs to identify the cause of the failure:

```bash
kubectl logs -n gion deployment/gion-api
```

The output contains the following error:

```
missing /app/.env
```

This indicates that the entrypoint script cannot find the configuration file required to start the application.

Inspect the handover note and application manifests:

```bash
cat /home/admin/HANDOVER.txt
cat /home/admin/app/deployment.yaml
cat /home/admin/app/secret.yaml
cat /home/admin/app/configmap.yaml
```

The handover note explains that the application requires `APP_TOKEN` from `/app/.env`. The entrypoint script checks that the file exists and validates the token before starting the HTTP server:

```
if [ ! -f /app/.env ]; then
  echo "missing /app/.env" >&2
  exit 1
fi

. /app/.env

if [ "${APP_TOKEN:-}" != "gion-ok-9c2e" ]; then
  echo "bad APP_TOKEN" >&2
  exit 1
fi
```

Inspect the secret's available keys:

```bash
kubectl get secret gion-dotenv -n gion -o jsonpath='{.data}'
echo
```

The secret contains the keys `.env` and `extra.conf`. The `.env` key contains the required `APP_TOKEN` and `APP_NAME` values.

However, the deployment mounts the secret using the following configuration:

```yaml
- name: dotenv
  mountPath: /app/.env
  subPath: env
```

The problem is that `subPath` references a secret key named `env`, which does not exist. The actual key is `.env`, so the required file is not mounted at `/app/.env`. As a result, the entrypoint script exits with the error `missing /app/.env`.

Fix the deployment manifest by changing `subPath: env` to `subPath: .env`.

Apply the updated manifest:

```bash
cd /home/admin/app
kubectl apply -f deployment.yaml
```

The deployment uses `hostNetwork: true` and binds the application to port `80`. Because the environment has a single Kubernetes node, the old and replacement pods cannot both bind to the same host port simultaneously. During the rolling update, the replacement pod remains pending because port `80` is occupied by the existing pod.

To release the port before starting the replacement pod, scale the deployment down to zero replicas:

```bash
kubectl scale deployment gion-api -n gion --replicas=0
kubectl rollout status deployment/gion-api -n gion
```

Verify that the pods have terminated:

```bash
kubectl get pods -n gion
```

Scale the deployment back up to one replica:

```bash
kubectl scale deployment gion-api -n gion --replicas=1
kubectl rollout status deployment/gion-api -n gion
```

Verify that the application pod is running and ready:

```bash
kubectl get pods -n gion
```

The Pod should show `1/1` in the READY column and `Running` in the STATUS column.

Rest the API endpoint:

```bash
curl -s http://127.0.0.1/health
```

The expected response is:

```bash
{"status":"ok","office":"gion-tickets"}
```