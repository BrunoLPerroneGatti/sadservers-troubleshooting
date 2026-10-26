## Problem

There are two pods: "logger" and "logshipper" living in the default namespace. Unfortunately, logshipper has an issue (crashlooping) and is forbidden to see what logger is trying to say. Could you help fix Logshipper?  
  
Do not change the K8S definition of the logshipper pod.

## Solution

Check the state of the pods:

```bash
kubectl get pods
```

The output shows that `logger` is Running, while `logshipper` is in `CrashLoopBackOff`.

Inspect the `logshipper` pod:

```bash
kubectl describe pod logshipper-597f84bf4f-fb5t2
```

The Pod uses the `logshipper-sa` ServiceAccount, and its container repeatedly exits with code `1`.

Inspect the logs from the previous container instance to identify the cause:

```
kubectl logs logshipper-597f84bf4f-fb5t2 --previous
```

The relevant error is:

```
Reason: Forbidden
User "system:serviceaccount:default:logshipper-sa"
cannot get resource "pods/log"
in the namespace "default"
```

The application is trying to read the logs of the `logger` pod through the Kubernetes API, but the API server denies the request because the ServiceAccount lacks the required permission.

Check the existing ClusterRole:

```bash
kubectl get clusterrole logshipper-cluster-role -o yaml
```

The ClusterRole contains the following rule:

```yaml
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  - pods
  - pods/log
  verbs:
  - list
```

The problem is that the ClusterRole only grants the `list` verb. Reading the logs of a specific Pod requires the `get` verb on the `pods/log` subresource.

Add the missing `get` verb to the existing ClusterRole without replacing its existing permissions:

```bash
kubectl patch clusterrole logshipper-cluster-role \
  --type='json' \
  -p='[{"op":"add","path":"/rules/0/verbs/-","value":"get"}]'
```

The rule now allows both `list` and `get` operations on the listed resources.

Check the pod's status and wait until the `logshipper` Pod becomes Ready and remains stable.

Run the required test:

```bash
kubectl get pods -l app=logshipper --no-headers -o json \
  | jq -r '.items[] | "\(.status.containerStatuses[0].ready)"'

# Expected output:
true
```
