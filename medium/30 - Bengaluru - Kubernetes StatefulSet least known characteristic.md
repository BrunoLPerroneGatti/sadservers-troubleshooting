## Problem

There's a Kubernetes cluster (created with "k3d") with two worker nodes and two pods on the node _k3d-cluster-agent-0_: a Deployment _demo-deployment-..._ and a StatefulSet _demo-statefulset-0_. Their manifests are identical except for the different kind of K8s resource.  
  
Make the node hosting the pods unavailable (it "goes down" or "crashes" without being deleted from k8s), for example with: docker stop k3d-mycluster-agent-0.  
  
After waiting for about a minute (_tolerationSeconds_ in the manifest is 30s, we shorten the K8S 5 minutes default so you don't have to wait so much, plus a grace period), both pods are marked as Terminating. While the Deployment pod is evicted and deployed onto the remaining available node _k3d-cluster-agent-1_, the StatefulSet _demo-statefulset-0_ is not (Why?).  
  
Make the StatefulSet pod _demo-statefulset-0_ run on the available node.

## Solution

Inspect the pods and the nodes where they are running:

```bash
kubectl get pods -o wide
```

Both `demo-deployment` and `demo-statefulset-0` are initially running on `k3d-mycluster-agent-0`.

Simulate a node failure by stopping the corresponding cluster:

```bash
docker stop k3d-mycluster-agent-0
```

Wait for Kubernetes to detect the node failure and inspect the pods again:

```bash
kubectl get pods -o wide
```

The Deployment creates a replacement pod on the healthy node `k3d-mycluster-agent-1`, while the original Deployment pod and `demo-statefulset-0` remain in the `Terminating` state.

Inspect the StatefulSet pod:

```bash
kubectl describe pod demo-statefulset-0
```

The output shows that the pod is still assigned to the unavailable node `k3d-mycluster-agent-0`. Its tolerations for `node.kubernetes.io/not-ready` and `node.kubernetes.io/unreachable` are set to 30 seconds, but the pod remains stuck terminating because the node cannot confirm that its containers have stopped.

This happen because deployment pods are generally interchangeable, so the Deployment controller can create a replacement on another healthy node.
StatefulSets, however, maintain stable pod identities, including predictable names such as `demo-statefulset-0`. Kubernetes avoids automatically creating a replacement with the same identity while the original pod may still be running on an unreachable node. This helps prevent two instances of the same StatefulSet identity from running simultaneously, which could cause data corruption or other consistency problems.

Force-delete the StatefulSet pod to remove its API object without waiting for the unavailable node to confirm termination:

```bash
kubectl delete pod demo-statefulset-0 --grace-period=0 --force
```

The StatefulSet controller detects the missing pod and creates a replacement. The scheduler places the new pod on the healthy node.

Verify the result:

```bash
kubectl get pods -o wide
```

The output should show `demo-statefulset-0` and the replacement Deployment pod both in the `Running` state on `k3d-mycluster-agent-1`.
