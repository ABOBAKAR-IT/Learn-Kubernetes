# Taints and Tolerations

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-scheduling/manifests/`](../labs/07-scheduling/manifests/) · Cluster setup: [`labs/07-scheduling`](../labs/07-scheduling/)

[← Pod Affinity](pod-affinity.md) | [README](README.md) | [Resource Requests and Limits →](resource-requests-limits.md)

---

## Idea

Affinity and selectors **attract** Pods to nodes. **Taints repel** them.

- A **taint** is a mark on a **node**: "do not put Pods here unless they accept this."
- A **toleration** is on a **Pod**: "I accept that taint."

```
Node W1  taint: dedicated=gpu:NoSchedule
              │
Pod A (no toleration)            ──► ✘ cannot be scheduled on W1
Pod B (tolerates dedicated=gpu)  ──► ✔ allowed on W1   (but not forced to go there)
```

> **A toleration only allows. It never attracts.** To make a Pod go to a node, add a `nodeSelector` or node affinity as well.

## Taint format

```
key=value:effect
```

```bash
kubectl taint nodes worker1 dedicated=gpu:NoSchedule        # add
kubectl taint nodes worker1 dedicated=gpu:NoSchedule-       # remove (note the trailing -)
kubectl taint nodes worker1 dedicated:NoSchedule-           # remove by key and effect
kubectl describe node worker1 | grep Taints
```

### The three effects

| Effect | New Pods | Pods already running |
|---|---|---|
| **`NoSchedule`** | Not scheduled (unless tolerated) | Left alone |
| **`PreferNoSchedule`** | Avoided if possible (soft) | Left alone |
| **`NoExecute`** | Not scheduled | **Evicted** unless they tolerate it |

## Toleration YAML

```yaml
spec:
  tolerations:
    - key: dedicated
      operator: Equal          # Equal (needs value) | Exists (any value)
      value: gpu
      effect: NoSchedule
```

```yaml
# tolerate any value of a key
- key: dedicated
  operator: Exists
  effect: NoSchedule

# tolerate EVERYTHING (this is what DaemonSets for node agents often do)
- operator: Exists

# tolerate a NoExecute taint for a limited time, then be evicted
- key: maintenance
  operator: Equal
  value: "true"
  effect: NoExecute
  tolerationSeconds: 30
```

| Field | Meaning |
|---|---|
| `key`, `value` | Must match the taint (`Equal`) |
| `operator` | `Equal` compares the value, `Exists` matches the key only |
| `effect` | Match a specific effect (empty matches all effects for that key) |
| `tolerationSeconds` | For `NoExecute` only: how long the Pod may stay after the taint appears |

## Taints Kubernetes adds by itself

| Taint | When | Effect |
|---|---|---|
| `node-role.kubernetes.io/control-plane` | On control-plane nodes in many clusters (kubeadm, kind) | `NoSchedule` |
| `node.kubernetes.io/unschedulable` | You ran `kubectl cordon` | `NoSchedule` |
| `node.kubernetes.io/not-ready` | Node is not Ready | `NoExecute` |
| `node.kubernetes.io/unreachable` | Control plane lost contact | `NoExecute` |
| `node.kubernetes.io/memory-pressure`, `disk-pressure`, `pid-pressure` | Node is short of that resource | `NoSchedule` |

Every Pod automatically gets **tolerations for `not-ready` and `unreachable` with `tolerationSeconds: 300`**. That is why a Pod on a dead node is replaced after about **5 minutes**:

```bash
kubectl get pod <any-pod> -o jsonpath='{.spec.tolerations}{"\n"}'
```

`kubectl cordon` is simply "add the `unschedulable` taint". `kubectl drain` = cordon + evict.

## The dedicated-nodes pattern

To reserve nodes for one workload you need **both** directions:

```
1. taint the nodes        → keeps everyone else OUT      kubectl taint nodes W1 dedicated=gpu:NoSchedule
2. label the nodes        → gives a handle to select     kubectl label node W1 dedicated=gpu
3. in the workload:
     toleration             → lets it IN
     nodeSelector/affinity  → makes it GO there
```

```yaml
spec:
  nodeSelector:
    dedicated: gpu
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
```

Without the `nodeSelector`, the workload may still run on other nodes. Without the taint, other workloads may also land on the dedicated nodes.

## Taints vs affinity

| | Taints and tolerations | Node affinity / selector |
|---|---|---|
| Direction | **Node repels** Pods | **Pod chooses** nodes |
| Who configures | Node owner (admin) | Pod owner (developer) |
| Default | Nodes open to everyone | n/a |
| Use for | Reserved nodes, draining, node problems | Placement preferences |

## Lab

Setup as in [Node Selector](node-selector.md): a multi-node cluster and two workers.

```bash
set -- $(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[*].metadata.name}')
W1=$1; W2=$2
cd labs/07-scheduling/manifests
kubectl describe node $W1 | grep Taints                    # Taints: <none>
```

**1. Taint a node: new Pods avoid it**

```bash
kubectl taint nodes $W1 dedicated=gpu:NoSchedule
kubectl describe node $W1 | grep Taints
kubectl create deployment plain --image=nginx:1.27 --replicas=6
kubectl get pods -l app=plain -o custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c    # none on $W1
```

**2. A toleration allows, it does not attract**

```bash
kubectl apply -f toleration-pod.yaml
kubectl get pod tolerates-gpu -o wide                      # may be on $W1 or elsewhere
```

**3. The dedicated-nodes pattern: both ends**

```bash
kubectl label node $W1 dedicated=gpu
kubectl apply -f dedicated-app.yaml
kubectl get pods -l app=gpu-app -o wide                    # both on $W1
```

**4. Remove the taint: existing Pods are not moved**

```bash
kubectl taint nodes $W1 dedicated=gpu:NoSchedule-
kubectl delete deployment plain
kubectl create deployment plain --image=nginx:1.27 --replicas=6
kubectl get pods -l app=plain -o custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c    # $W1 gets some again
```

**5. `NoExecute` evicts running Pods**

```bash
kubectl delete deployment plain
kubectl label node $W2 lab=noexec
kubectl apply -f noexecute-demo.yaml                       # two Pods pinned to $W2
kubectl get pods -o wide | grep noexec
kubectl taint nodes $W2 maintenance=true:NoExecute
kubectl get pods -w | grep noexec                          # noexec-plain disappears at once; noexec-30s after about 30 seconds
kubectl taint nodes $W2 maintenance=true:NoExecute-
```

These are bare Pods, so nothing recreates them. A Deployment's Pods would be rescheduled to another node.

**6. Cordon is a taint**

```bash
kubectl cordon $W1
kubectl describe node $W1 | grep -A1 Taints                # node.kubernetes.io/unschedulable:NoSchedule
kubectl uncordon $W1
```

**7. Built-in tolerations on every Pod**

```bash
kubectl get pod tolerates-gpu -o jsonpath='{range .spec.tolerations[*]}{.key}{" "}{.effect}{" "}{.tolerationSeconds}{"\n"}{end}'
```

**Cleanup**

```bash
kubectl delete -f toleration-pod.yaml -f dedicated-app.yaml -f noexecute-demo.yaml --ignore-not-found
kubectl delete deployment plain --ignore-not-found
kubectl label node $W1 dedicated- ; kubectl label node $W2 lab-
```

## Gotchas

- **A toleration does not attract.** Add a `nodeSelector` or affinity so the workload actually goes to the tainted nodes.
- **Taint and label together:** a taint alone keeps others out but does not tell your workload where to go.
- **`NoExecute` evicts** running Pods immediately, so be careful tainting a node in use.
- **Control-plane nodes** are tainted in kind and kubeadm clusters (minikube leaves them open). DaemonSets for agents need a toleration to run there.
- Removing a taint needs the **exact** key and effect plus a trailing `-`.
- Unmatched Pods show `0/N nodes are available: N node(s) had untolerated taint {...}` in their Events.

## Check yourself

1. Who sets a taint, and who sets a toleration?
2. What are the three taint effects?
3. Does a toleration make a Pod go to the tainted node?
4. How do you reserve a set of nodes for one workload?
5. Why are Pods on a dead node replaced after about 5 minutes?
6. What does `kubectl cordon` do in terms of taints?

<details>
<summary>Answers</summary>

1. A taint is set on a node (admin). A toleration is set on a Pod (developer)
2. `NoSchedule`, `PreferNoSchedule`, `NoExecute`
3. No. It only permits it. Use a nodeSelector or affinity to attract
4. Taint the nodes, label them, and give the workload a matching toleration plus a nodeSelector (or affinity) for that label
5. Pods tolerate the `not-ready` and `unreachable` NoExecute taints for 300 seconds by default
6. It adds the `node.kubernetes.io/unschedulable:NoSchedule` taint

</details>

---

[← Pod Affinity](pod-affinity.md) | [README](README.md) | [Resource Requests and Limits →](resource-requests-limits.md)