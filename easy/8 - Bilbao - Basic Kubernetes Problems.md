## Problem

There's a Kubernetes Deployment with an Nginx pod and a Load Balancer declared in the _manifest.yml_ file. The pod is not coming up. Fix it so that you can access the Nginx container through the Load Balancer.

Running ``curl 10.43.216.196`` returns the default Nginx Welcome page.
## Solution

 Inspect the manifest and Pod status

```bash
cat manifest.yml 
```

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/library/nginx:1.27.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 2000Mi
            cpu: 100m
          requests:
            cpu: 100m
            memory: 2000Mi
      nodeSelector:
        disk: ssd

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  clusterIP: 10.43.216.196
  type: LoadBalancer
```

```
kubectl get pods
kubectl describe pod -l app=nginx
```

The pod was stuck in `pending`. The first scheduling error was:

```
0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```

The manifest contained this `nodeSelector`:

```
nodeSelector:
  disk: ssd
```

The only node did not have the required `disk=ssd` label.

Edit the manifest and remove the following section from the Deployment's Pod specification:

```
nodeSelector:
  disk: ssd
```

Apply the changes:

```
kubectl apply -f manifest.yml
```

This creates a new ReplicaSet because the Pod template changed.


After applying the manifest, both the old and new ReplicaSets had one desired replica. Deleting the old Pod alone did not solve the problem because its ReplicaSet recreated it.

Scale the Deployment down to zero and back to one to allow the rollout to converge:

```
kubectl scale deployment/nginx-deployment --replicas=0
kubectl scale deployment/nginx-deployment --replicas=1
```


After removing the node selector, the Pod was still `Pending`. The new scheduler error was:

```bash
0/1 nodes are available: 1 Insufficient memory.
```
The manifest requested and limited the container to `2000Mi` of memory:

```yaml
resources:
  limits:
    memory: 2000Mi
    cpu: 100m
  requests:
    cpu: 100m
    memory: 2000Mi
```

The node did not have enough available memory to satisfy the request.
Edit the manifest remove both the memory request and memory limit.

```yaml
resources:
  limits:
    cpu: 100m
  requests:
    cpu: 100m
```

Apply the updated manifest:

```
kubectl apply -f manifest.yml
```
