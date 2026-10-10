# Node Affinity

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-scheduling/manifests/`](../labs/07-scheduling/manifests/) · Cluster setup: [`labs/07-scheduling`](../labs/07-scheduling/)

[← Node Selector](node-selector.md) | [README](README.md) | [Pod Affinity →](pod-affinity.md)

---

## Idea

**Node affinity** is the more expressive version of `nodeSelector`. It adds **operators** (In, NotIn, Exists...), **OR** between rules, and **soft preferences** ("try to, but not required").

```
nodeSelector:   disktype = ssd                      (exact match, required)
nodeAffinity:   zone In [a, b]   AND   gpu Exists    (required)
                prefer disktype = ssd (weight 80)    (nice to have)
```

## The two kinds

The field names look long, and each part has a meaning:

```
requiredDuringSchedulingIgnoredDuringExecution
└─ required ──────┘ └ when scheduling ┘ └ if labels change later, running Pods stay ┘

preferredDuringSchedulingIgnoredDuringExecution
└─ preferred ─────┘
```

| Kind | Behavior | If no node fits |
|---|---|---|
| **required...** | **Hard rule** (a filter) | Pod stays `Pending` |
| **preferred...** | **Soft rule** (adds to a node's score) | Pod is still scheduled somewhere else |

"IgnoredDuringExecution" means that if a node's labels change **after** the Pod is running, the Pod is **not** evicted.

## YAML explained

**Required**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: zone-a-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: zone
                operator: In
                values: ["a"]
  containers:
    - name: app
      image: nginx:1.27
```

**Preferred, with weights**

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80                     # 1 to 100
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
        - weight: 20
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["b"]
```

Each matching preference **adds its weight** to the node's score. The scheduler picks the highest total (other scoring rules also count).

### Operators

| Operator | Meaning |
|---|---|
| `In` | Label value is one of the values |
| `NotIn` | Label value is none of the values (repel) |
| `Exists` | The label key exists (any value) |
| `DoesNotExist` | The label key is absent |
| `Gt`, `Lt` | Label value (an integer) is greater or less than |

### AND and OR (the part people get wrong)

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:           # items in this list are ORed
    - matchExpressions:        #   items in this list are ANDed
        - { key: zone,    operator: In,     values: [a, b] }
        - { key: gpu,     operator: Exists }
    - matchExpressions:        # OR this whole term instead
        - { key: special, operator: Exists }
```

| Level | Combined with |
|---|---|
| Several **`nodeSelectorTerms`** | **OR** (any one term matches) |
| Several **`matchExpressions`** in one term | **AND** (all must match) |
| `values` within one expression | **OR** (any value) |

### Combining with `nodeSelector`

If you set both `nodeSelector` and node affinity, the node must satisfy **both**.

### `matchFields`

Match node fields instead of labels (for example the node name):

```yaml
matchFields:
  - key: metadata.name
    operator: In
    values: ["worker1"]
```

## Common patterns

```yaml
# Anti-affinity for nodes: avoid spot nodes
- key: node.kubernetes.io/lifecycle
  operator: NotIn
  values: ["spot"]

# Only Linux ARM nodes
- key: kubernetes.io/arch
  operator: In
  values: ["arm64"]

# Prefer cheaper nodes, but run anywhere if they are full
preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    preference:
      matchExpressions:
        - { key: pricing, operator: In, values: ["spot"] }
```

## nodeSelector vs node affinity

| | `nodeSelector` | Node affinity |
|---|---|---|
| Syntax | Simple map | Verbose, structured |
| Operators | Equality only | In, NotIn, Exists, DoesNotExist, Gt, Lt |
| OR | No | Yes (multiple terms) |
| Soft preferences | No | **Yes** |
| Use when | One or two exact labels | Anything more |

## Lab

Setup as in [Node Selector](node-selector.md): a multi-node cluster and two workers.

```bash
set -- $(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[*].metadata.name}')
W1=$1; W2=$2
kubectl label node $W1 zone=a disktype=ssd
kubectl label node $W2 zone=b
cd labs/07-scheduling/manifests
```

**1. Required: lands only on `zone=a`**

```bash
kubectl apply -f affinity-required.yaml
kubectl get pod zone-a-pod -o wide                        # on $W1
```

**2. An impossible requirement, then fix it by labelling**

```bash
kubectl apply -f affinity-impossible.yaml                 # asks for zone In [c]
kubectl get pod zone-c-pod                                # Pending
kubectl describe pod zone-c-pod | grep -A3 Events         # didn't match Pod's node affinity/selector
kubectl label node $W2 zone=c --overwrite                 # now a node qualifies
kubectl get pod zone-c-pod -o wide                        # scheduled on $W2
kubectl label node $W2 zone=b --overwrite                 # put it back
```

**3. Preferred: most Pods go to the SSD node, but not all**

```bash
kubectl apply -f affinity-preferred.yaml                  # 6 replicas, prefer disktype=ssd (80) and zone b (20)
kubectl get pods -l app=pref -o custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c
```

Most Pods land on `$W1` (the SSD node), and some may land elsewhere, because scoring also considers spreading and other factors. A preference is not a rule.

**4. "IgnoredDuringExecution": running Pods are not evicted**

```bash
kubectl label node $W1 zone-                              # remove the label zone-a-pod required
kubectl get pod zone-a-pod -o wide                        # still Running on $W1
kubectl delete pod zone-a-pod
kubectl apply -f affinity-required.yaml
kubectl get pod zone-a-pod                                # now Pending: no node has zone=a
```

**Cleanup**

```bash
kubectl delete -f affinity-required.yaml -f affinity-impossible.yaml -f affinity-preferred.yaml --ignore-not-found
kubectl label node $W1 zone- disktype- ; kubectl label node $W2 zone-
```

## Gotchas

- **Required** can leave Pods `Pending`. **Preferred** never blocks scheduling.
- It is `required...IgnoredDuringExecution`: changing node labels later does **not** move running Pods.
- Two terms are **OR**, two expressions in a term are **AND**. One missing indentation level flips the meaning.
- Operator values are **strings**. For `Gt` and `Lt` the label value must be an integer.
- Do not use affinity to **repel** Pods from a node for exclusivity. That is the job of [taints](taints-tolerations.md).
- Combining a `nodeSelector` and affinity requires **both** to match.

## Check yourself

1. What is the difference between required and preferred node affinity?
2. What does "IgnoredDuringExecution" mean?
3. How do you express "zone is a or b, and the node has a GPU label"?
4. What happens when you list two `nodeSelectorTerms`?
5. Which operator keeps Pods away from nodes with a certain label value?

<details>
<summary>Answers</summary>

1. Required is a hard filter (Pod stays Pending if nothing fits). Preferred only adds to a node's score
2. If node labels change after the Pod is running, the Pod is not evicted. The rule applies only at scheduling time
3. One term with two expressions: `zone In [a, b]` and `gpu Exists`
4. They are ORed: a node matching either term qualifies
5. `NotIn` (or `DoesNotExist` for a missing key)

</details>

---

[← Node Selector](node-selector.md) | [README](README.md) | [Pod Affinity →](pod-affinity.md)