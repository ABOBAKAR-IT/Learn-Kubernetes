# kube-controller-manager

> Status: 📘 Notes ready

## Idea

The **controller manager** runs many **controllers** in one process. Each controller is a **control loop**:

```
        ┌─────────────────────────────────────────┐
        ▼                                         │
  OBSERVE actual state   ─►  COMPARE with desired state  ─►  ACT to close the gap
  (watch the API server)                                    (create/delete/update objects)
```

This is the reconciliation loop from the basics. Controllers only talk to the **API server**. They never touch containers directly.

## Controllers you already know

| Controller | Watches | Does |
|---|---|---|
| **Deployment** | Deployments | Creates and scales ReplicaSets, runs rolling updates |
| **ReplicaSet** | ReplicaSets | Creates or deletes Pods to match `replicas` |
| **StatefulSet / DaemonSet / Job / CronJob** | Their own objects | Manage Pods their own way |
| **Node** | Nodes | Detects unreachable nodes, marks them `NotReady`, taints them so Pods are evicted |
| **EndpointSlice** | Services and Pods | Keeps the list of Pod IPs behind each Service up to date |
| **Namespace** | Namespaces | Deletes everything inside a namespace when it is deleted |
| **PersistentVolume** | PVCs and PVs | Binds claims to volumes |
| **ServiceAccount** | Namespaces | Creates the `default` ServiceAccount in each |
| **Garbage collector** | `ownerReferences` | Deletes objects whose owner is gone |
| **HPA** | Metrics | Changes replicas of Deployments |

## Pseudo-code of a controller

```python
while True:
    desired = api.get(ReplicaSet).spec.replicas       # 3
    actual  = api.count_pods(matching_labels)         # 2
    if actual < desired:  api.create_pods(desired - actual)
    if actual > desired:  api.delete_pods(actual - desired)
    wait_for_next_event()                             # via watch
```

## Ownership: how the garbage collector knows what to delete

Objects created by a controller carry an **`ownerReferences`** field:

```
Deployment web
   └─ owns ─► ReplicaSet web-7d9f8
                 └─ owns ─► Pod web-7d9f8-abcde
```

```bash
kubectl get pod <pod> -o jsonpath='{.metadata.ownerReferences[0].kind}{"/"}{.metadata.ownerReferences[0].name}{"\n"}'
```

Delete the owner and the dependents go too (**cascading delete**). Keep the dependents with `--cascade=orphan`.

## Node failure: what the node controller does

1. A node stops sending heartbeats (its Lease is not renewed).
2. After a grace period (about 40 seconds by default) the node becomes `NotReady`/`Unknown`.
3. The node gets a taint (`node.kubernetes.io/unreachable`).
4. Pods get a default toleration of **300 seconds** for that taint. After it, they are evicted.
5. ReplicaSet controllers see fewer Pods and create replacements, and the scheduler places them on healthy nodes.

So a dead node's Pods are replaced after roughly **5 to 6 minutes** unless you tune it.

## Leader election

Control planes run several copies of the controller manager (and scheduler) for availability, but **only one is active**. They compete for a **Lease** object:

```bash
kubectl -n kube-system get leases
kubectl -n kube-system get lease kube-controller-manager -o yaml | grep holderIdentity
```

## cloud-controller-manager

On cloud clusters a separate **cloud-controller-manager** holds the cloud-specific loops: node lifecycle (removing a node when its VM is gone), routes, and creating **load balancers** for `type: LoadBalancer` Services. This is how EKS turns a Service into an AWS load balancer.

## Lab

```bash
# 1. See self-healing in action
kubectl create deployment demo --image=nginx:1.27 --replicas=3
kubectl get events -w &                       # watch events in the background
kubectl delete pod -l app=demo --wait=false   # kill all three
kubectl get pods                              # replacements already appearing
kill %1

# 2. Follow the ownership chain
kubectl get deploy,rs,pods -l app=demo
kubectl get pod -l app=demo -o jsonpath='{.items[0].metadata.ownerReferences[0].kind}{"\n"}'   # ReplicaSet
kubectl get rs -l app=demo -o jsonpath='{.items[0].metadata.ownerReferences[0].kind}{"\n"}'     # Deployment

# 3. Cascading delete vs orphan
kubectl delete rs -l app=demo --cascade=orphan
kubectl get pods -l app=demo                  # Pods still exist, now orphaned
kubectl delete deployment demo
kubectl delete pod -l app=demo --ignore-not-found

# 4. Leader election
kubectl -n kube-system get leases
```

## Gotchas

- Controllers are **eventually consistent**: there is a short delay between a change and the fix.
- Deleting a ReplicaSet-owned Pod "does nothing" lasting: the controller makes another.
- The controller manager being down means **no self-healing and no rollouts**, but existing Pods keep running.

## Check yourself

1. Describe a control loop in three words.
2. Which controller creates Pods when you scale a ReplicaSet?
3. What is `ownerReferences` used for?
4. How long until Pods on a dead node are replaced, by default?
5. Why does only one controller manager replica act at a time?

<details>
<summary>Answers</summary>

1. Observe, compare, act
2. The ReplicaSet controller (inside the controller manager)
3. Linking dependents to their owner so the garbage collector can cascade deletes
4. Roughly 5 to 6 minutes (grace period plus the default 300 second toleration)
5. Leader election through a Lease prevents two copies acting at once

</details>