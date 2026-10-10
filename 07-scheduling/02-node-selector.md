# Node Selector

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-scheduling/manifests/`](../labs/07-scheduling/manifests/) · Cluster setup: [`labs/07-scheduling`](../labs/07-scheduling/)

[← Labels and Selectors](labels-selectors.md) | [README](README.md) | [Node Affinity →](node-affinity.md)

---

## Idea

**`nodeSelector`** is the simplest way to say **"run this Pod only on nodes that have these labels."**

```
Pod:  nodeSelector: { disktype: ssd }
                 │
Scheduler:  Node A  disktype=ssd   ✔ eligible
            Node B  disktype=hdd   ✘ filtered out
            Node C  (no label)     ✘ filtered out
```

It is a **hard requirement** (a filter): if no node matches, the Pod stays **Pending**.

## Two steps

```bash
# 1. label the node
kubectl label node <node> disktype=ssd

# 2. ask for it in the Pod
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: app
      image: nginx:1.27
```

- **Several entries are ANDed.** The node must have **all** the labels.
- It works in Deployments and other controllers inside `spec.template.spec`.

## Using built-in labels

You do not have to add labels yourself:

```yaml
nodeSelector:
  kubernetes.io/os: linux
  kubernetes.io/arch: arm64               # run on Graviton / Apple silicon nodes
  topology.kubernetes.io/zone: eu-west-1a # stay in one zone
  node.kubernetes.io/instance-type: m5.large
```

## What it cannot do

| Need | `nodeSelector`? |
|---|---|
| "Node must have label A **and** B" | Yes |
| "Label A **or** B" | No |
| "Label **not** equal to X" | No |
| "**Prefer** SSD but allow others" | No (always required) |
| Rules based on **other Pods** | No |

For those, use [node affinity](node-affinity.md) or [pod affinity](pod-affinity.md).

## Related: `nodeName`

```yaml
spec:
  nodeName: worker1       # bypasses the scheduler entirely
```

`nodeName` assigns the Pod to **one named node** directly. It skips all filters, so taints and resource checks do not apply. Avoid it in real workloads: if the node is down or full, the Pod just fails.

## Keep it from being forgotten: it only affects scheduling

`nodeSelector` is checked **when the Pod is scheduled**. If you later remove the label from the node, **running Pods stay** where they are. (In the API this is the `IgnoredDuringExecution` part you will meet in node affinity.)

## Lab

You need a multi-node cluster. See [`labs/07-scheduling`](../labs/07-scheduling/) (`minikube start -p sched --nodes 3` or the kind config). Pick two worker nodes:

```bash
kubectl get nodes
set -- $(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[*].metadata.name}')
W1=$1; W2=$2; echo "$W1 $W2"
cd labs/07-scheduling/manifests
```

**1. A Pod nobody can run**

```bash
kubectl apply -f nodeselector-pod.yaml
kubectl get pod ssd-pod                                   # Pending
kubectl describe pod ssd-pod | grep -A3 Events            # didn't match Pod's node affinity/selector
```

**2. Label a node and watch the Pod land**

```bash
kubectl label node $W1 disktype=ssd
kubectl get pod ssd-pod -o wide                           # scheduled on $W1, no restart needed
```

**3. A Deployment follows the label**

```bash
kubectl apply -f nodeselector-deploy.yaml
kubectl get pods -l app=ssd-app -o wide                   # all 4 on $W1
kubectl label node $W2 disktype=ssd
kubectl scale deployment ssd-app --replicas=8
kubectl get pods -l app=ssd-app -o custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c   # spread over $W1 and $W2
```

**4. The label is only checked at scheduling time**

```bash
kubectl label node $W1 disktype-                          # remove the label
kubectl get pods -l app=ssd-app -o wide                   # existing Pods stay on $W1
kubectl rollout restart deployment ssd-app
kubectl get pods -l app=ssd-app -o wide                   # new Pods only on $W2
```

**5. Bypass with `nodeName`** (to see the difference)

```bash
kubectl run manual --image=nginx:1.27 --overrides="{\"spec\":{\"nodeName\":\"$W1\"}}"
kubectl get pod manual -o wide                            # on $W1, no scheduler involved
```

**Cleanup**

```bash
kubectl delete pod ssd-pod manual
kubectl delete deployment ssd-app
kubectl label node $W1 disktype- ; kubectl label node $W2 disktype-
```

## Gotchas

- A typo in the label (key or value) leaves the Pod `Pending` with no other error.
- Selecting by a label **no node has** is the number one cause of "Pending" Pods in this area.
- Values are strings: write `nodeSelector: { gpu: "true" }`.
- Labelling nodes by hand is lost when nodes are replaced (autoscaling). Put labels in the **node group or machine template** (for example EKS managed node groups) instead.
- `nodeSelector` gives no fallback. If you want "prefer", use node affinity.

## Check yourself

1. What happens to a Pod whose `nodeSelector` matches no node?
2. If a `nodeSelector` has two labels, must a node have both?
3. Name two built-in labels you can use in a `nodeSelector`.
4. If you remove the label from a node, what happens to Pods already running there?
5. Why avoid `nodeName`?

<details>
<summary>Answers</summary>

1. It stays `Pending`, and the Events say it did not match the node selector or affinity
2. Yes, all entries must match (AND)
3. For example `kubernetes.io/os`, `kubernetes.io/arch`, `topology.kubernetes.io/zone`
4. Nothing: they keep running. Only new scheduling decisions are affected
5. It skips the scheduler and every filter, so a down or full node just makes the Pod fail

</details>

---

[← Labels and Selectors](labels-selectors.md) | [README](README.md) | [Node Affinity →](node-affinity.md)