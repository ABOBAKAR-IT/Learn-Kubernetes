# kube-scheduler

> Status: 📘 Notes ready

## Idea

The **scheduler** decides **which node** each new Pod runs on. It does **not** start the Pod. The kubelet does that.

- It watches for Pods whose `spec.nodeName` is empty.
- It picks a node and **binds** the Pod by writing `nodeName` through the API server.

## How it decides: filter, then score

```
Pod needs a node
   │
   ▼  1. FILTER: remove nodes that cannot run it
   │     - not enough CPU/memory (compares requests to allocatable)
   │     - nodeSelector / node affinity does not match
   │     - taints the Pod does not tolerate
   │     - volume or port conflicts
   ▼  2. SCORE: rank the nodes that are left
   │     - spread Pods of the same app
   │     - balanced resource use
   │     - image already on the node
   │     - preferred affinity rules
   ▼  3. PICK the highest score (ties broken randomly)
   ▼  4. BIND: set spec.nodeName
```

If every node is filtered out, the Pod stays **Pending** and the reason appears in its events.

## Important facts

| Fact | Why it matters |
|---|---|
| It uses resource **requests**, not live usage and not limits | A Pod with no requests can be packed onto a busy node |
| It decides **once**, at creation time | It does not move running Pods when a better node appears |
| `kubelet` starts the Pod after binding | Scheduler only chooses |
| You can run **extra schedulers** | Set `spec.schedulerName` on a Pod |

## Lab

```bash
kubectl get nodes
```

**1. Normal scheduling**

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=4
kubectl get pods -o wide                       # note the NODE column
kubectl describe pod <web-pod> | grep -A6 Events    # "Scheduled ... default-scheduler"
```

**2. Impossible request: stays Pending**

```yaml
# big-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "100"        # 100 CPUs, no node has that
```

```bash
kubectl apply -f big-pod.yaml
kubectl get pod big-pod                         # Pending
kubectl describe pod big-pod | grep -A4 Events  # FailedScheduling: Insufficient cpu
```

**3. No matching node label: Pending**

```bash
kubectl run picky --image=nginx:1.27 --overrides='{"spec":{"nodeSelector":{"disktype":"ssd"}}}'
kubectl describe pod picky | grep -A4 Events    # didn't match Pod's node affinity/selector
kubectl label node <a-node> disktype=ssd        # fix: label a node
kubectl get pod picky -o wide                   # scheduled now, no restart needed
```

**4. Bypass the scheduler**

```yaml
# manual-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: <a-node-name>      # set by hand: the scheduler is skipped
  containers:
    - name: app
      image: nginx:1.27
```

**Cleanup**

```bash
kubectl delete deployment web
kubectl delete pod big-pod picky manual-pod --ignore-not-found
kubectl label node <a-node> disktype-
```

## Gotchas

- `Pending` plus `FailedScheduling` events means a **scheduling** problem, not an image or app problem.
- Setting `nodeName` yourself skips filters like taints and resource checks.
- A busy cluster can still have unevenly loaded nodes. Rebalancing needs tools like the **descheduler**, or topology spread constraints.

## Check yourself

1. What does the scheduler write when it picks a node?
2. Name the two phases of scheduling.
3. Does the scheduler look at live CPU usage?
4. A Pod is Pending with `Insufficient memory`. Where do you look first?
5. Who actually starts the containers after scheduling?

<details>
<summary>Answers</summary>

1. The Pod's `spec.nodeName` (a binding)
2. Filtering, then scoring
3. No. It compares the Pod's requests with each node's allocatable resources
4. `kubectl describe pod <name>`, the Events section
5. The kubelet on the chosen node

</details>