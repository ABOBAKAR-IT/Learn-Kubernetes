# Pod Affinity and Anti-Affinity

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-scheduling/manifests/`](../labs/07-scheduling/manifests/) · Cluster setup: [`labs/07-scheduling`](../labs/07-scheduling/)

[← Node Affinity](node-affinity.md) | [README](README.md) | [Taints and Tolerations →](taints-tolerations.md)

---

## Idea

Node affinity looks at **node labels**. **Pod affinity** looks at **which other Pods are already running** and places a Pod **near** (affinity) or **away from** (anti-affinity) them.

| Goal | Use |
|---|---|
| Run the web Pods on the same node as their cache (low latency) | **podAffinity** |
| Do not put two replicas of the same app on one node (availability) | **podAntiAffinity** |
| Spread replicas evenly over nodes or zones | **topologySpreadConstraints** (usually the better tool) |

## topologyKey: what does "near" mean?

Every rule has a **`topologyKey`**, a **node label** that defines the **domain**:

| `topologyKey` | "Same domain" means | "Different domain" means |
|---|---|---|
| `kubernetes.io/hostname` | The **same node** | Another node |
| `topology.kubernetes.io/zone` | The **same availability zone** | Another zone |
| `topology.kubernetes.io/region` | The same region | Another region |

```
affinity,      key=hostname  →  same node as the target Pods
anti-affinity, key=hostname  →  a different node from every target Pod
anti-affinity, key=zone      →  a different zone from every target Pod
```

## YAML explained

**Affinity: web next to cache**

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["cache"]
          topologyKey: kubernetes.io/hostname
```

*"Schedule me only on a node that already runs a Pod labeled `app=cache`."*

**Anti-affinity: one replica per node**

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: spread-hard
          topologyKey: kubernetes.io/hostname
```

*"Never schedule me on a node that already runs a Pod labeled `app=spread-hard`."* The Pod repels its own replicas.

**Soft anti-affinity: prefer different zones**

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: topology.kubernetes.io/zone
```

### Fields

| Field | Meaning |
|---|---|
| `labelSelector` | Which **Pods** to be near or away from |
| `topologyKey` | What "near" means (node, zone, region) |
| `namespaces` / `namespaceSelector` | Which namespaces to look in (default: the Pod's own) |
| `required...` | Hard rule, Pod stays `Pending` if unmet |
| `preferred...` (with `weight`) | Soft rule, adds to node score |

The same "IgnoredDuringExecution" rule applies: running Pods are not moved when things change.

## The trap with required anti-affinity

With **required** anti-affinity per node and **more replicas than nodes**, the extra Pods stay **Pending forever**. Rolling updates can also get stuck: the **surge Pod** needs a free node.

```
5 replicas, required anti-affinity per node, 3 eligible nodes
→ 3 Pods run, 2 stay Pending
```

Use **preferred** anti-affinity, or better, topology spread.

## topologySpreadConstraints: spread evenly

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: spread
      nodeAffinityPolicy: Honor
      nodeTaintsPolicy: Honor
```

| Field | Meaning |
|---|---|
| `maxSkew` | The maximum allowed **difference** in Pod count between the fullest and emptiest domain |
| `topologyKey` | The domain (node, zone) |
| `whenUnsatisfiable` | `DoNotSchedule` (hard) or `ScheduleAnyway` (soft: prefer the best spread) |
| `labelSelector` | Which Pods are counted |
| `minDomains` | Require at least this many domains |

**Skew** = Pods in the fullest domain minus Pods in the emptiest domain. With `maxSkew: 1`, counts like 2/1/1 are fine and 3/1/0 is not.

### Anti-affinity vs topology spread

| | Pod anti-affinity | Topology spread |
|---|---|---|
| Idea | "Never **together**" | "Keep the counts **balanced**" |
| More replicas than domains | Extra Pods **Pending** (if required) | Placed **evenly**, several per domain |
| Several levels (zone **and** node) | Awkward | Several constraints, easy |
| Good for | Strict isolation (one per node) | **Spreading replicas for availability** |

A common production setup is a **zone** spread (hard or soft) plus a **node** spread (soft).

## Cost: use with care

Pod affinity and anti-affinity make the scheduler compare against **other Pods**. In very large clusters this is noticeably slower than node-based rules. Prefer topology spread for spreading, and use affinity only where you need it.

## Lab

A multi-node cluster is needed (see [`labs/07-scheduling`](../labs/07-scheduling/)).

```bash
cd labs/07-scheduling/manifests
kubectl get nodes
```

**1. Affinity: web follows cache**

```bash
kubectl apply -f cache-web-affinity.yaml
kubectl get pods -l 'app in (cache,web)' -o wide          # the 3 web Pods share the cache Pod's node
```

Where does the cache land? Anywhere. Then `web` is pulled to that node. Scale web up and see that all stay together:

```bash
kubectl scale deployment web --replicas=6
kubectl get pods -l app=web -o custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c    # one node only
```

**2. Required anti-affinity: Pods that cannot all fit**

```bash
kubectl apply -f anti-affinity.yaml                       # 5 replicas, one per node at most
kubectl get pods -l app=spread-hard -o wide               # as many Running as eligible nodes, the rest Pending
kubectl describe pod -l app=spread-hard | grep -m1 -A2 "didn't match pod anti-affinity"
```

**3. Topology spread: even placement**

```bash
kubectl apply -f topology-spread.yaml                     # 4 replicas, maxSkew 1 per node
kubectl get pods -l app=spread -o custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c
kubectl scale deployment spread --replicas=7
kubectl get pods -l app=spread -o custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c    # counts differ by at most 1
```

**4. See the difference**

Compare step 2 (extra Pods stuck Pending) with step 3 (all Pods placed, balanced). That is the practical difference between the two tools.

**Cleanup**

```bash
kubectl delete -f cache-web-affinity.yaml -f anti-affinity.yaml -f topology-spread.yaml
```

## Gotchas

- **`topologyKey` is a node label.** If nodes lack that label, the rule cannot work (zone labels need a cloud or manual labelling).
- **Required anti-affinity with replicas > nodes** leaves Pods `Pending`.
- Selectors are evaluated against **running** Pods only at scheduling time. Later changes do not move Pods.
- Tainted nodes (such as a control-plane node) are counted as spread **domains** unless `nodeTaintsPolicy: Honor` is set. That can make a `DoNotSchedule` constraint hold Pods back.
- Anti-affinity should select the Pod's **own** labels (so it repels replicas), not unrelated labels.
- Affinity across many Pods is **expensive** in large clusters.

## Check yourself

1. What does `topologyKey: kubernetes.io/hostname` mean in an anti-affinity rule?
2. What happens with 5 replicas, required anti-affinity per node, and 3 nodes?
3. What does `maxSkew: 1` allow? Is 3/1/0 acceptable?
4. When would you pick topology spread over anti-affinity?
5. Why can required anti-affinity block a rolling update?

<details>
<summary>Answers</summary>

1. Different nodes: no two matching Pods may share a node
2. 3 Pods run, 2 stay Pending
3. A difference of at most 1 between the fullest and emptiest domain. 3/1/0 is not acceptable (skew 3)
4. When you want replicas spread evenly and the replica count may exceed the number of domains
5. The new surge Pod needs a node that does not already run a matching Pod, and there may be none free

</details>

---

[← Node Affinity](node-affinity.md) | [README](README.md) | [Taints and Tolerations →](taints-tolerations.md)