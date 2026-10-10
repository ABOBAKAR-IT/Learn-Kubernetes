# Module 07: Scheduling

> Control where Pods run: labels, selectors, affinity, taints and tolerations, and resource requests and limits.

**Notes ready:** 6/6 topics

## Topic directory

| Topic | File | Question it answers | Status |
|---|---|---|---|
| **Labels and Selectors** | [`labels-selectors.md`](labels-selectors.md) | How do objects find each other? | 📘 |
| **Node Selector** | [`node-selector.md`](node-selector.md) | How do I run a Pod only on certain nodes? | 📘 |
| **Node Affinity** | [`node-affinity.md`](node-affinity.md) | How do I express richer or softer node rules? | 📘 |
| **Pod Affinity** | [`pod-affinity.md`](pod-affinity.md) | How do I keep Pods together, or spread them apart? | 📘 |
| **Taints and Tolerations** | [`taints-tolerations.md`](taints-tolerations.md) | How do nodes repel Pods and stay reserved? | 📘 |
| **Resource Requests and Limits** | [`resource-requests-limits.md`](resource-requests-limits.md) | How much does a Pod reserve, and what happens when it overuses? | 📘 |

YAML for the labs: [`labs/07-scheduling/manifests/`](../labs/07-scheduling/manifests/). Cluster setup and a combined scenario: [`labs/07-scheduling`](../labs/07-scheduling/).

## The big idea

Scheduling answers one question for each new Pod: **which node?** The scheduler **filters out** nodes that cannot run it, then **scores** the rest and picks the best (see [the scheduler](../02-kubernetes-architecture/control-plane/scheduler.md)). Every feature in this module is a way to influence that decision.

```
All nodes
   │  FILTER (hard rules)
   │    ├─ resource requests fit?            → requests
   │    ├─ nodeSelector / required affinity  → node labels
   │    ├─ taints tolerated?                 → taints
   │    └─ required pod (anti)affinity, hard spread rules
   ▼
Eligible nodes
   │  SCORE (soft rules)
   │    ├─ preferred node affinity (weights)
   │    ├─ preferred pod (anti)affinity, soft spread
   │    └─ built-in spreading, image already present...
   ▼
Best node  →  Pod is bound there
```

**Everything is label-driven**: you label nodes and Pods, and rules refer to those labels.

## Attract, repel, and size

| Direction | Tool | Set by |
|---|---|---|
| **Attract to nodes** | `nodeSelector`, node affinity | The Pod (developer) |
| **Attract to / repel from other Pods** | Pod affinity, anti-affinity, topology spread | The Pod (developer) |
| **Repel from nodes** | Taints (Pods need tolerations) | The node (admin) |
| **Size and importance** | Requests, limits, QoS, PriorityClass | The Pod and the namespace |

## Which tool do I use?

| I want to... | Use |
|---|---|
| Run on nodes with one or two exact labels | **nodeSelector** |
| Run on nodes by richer rules, or only *prefer* some | **Node affinity** |
| Put Pods next to other Pods (cache, same node or zone) | **Pod affinity** |
| Keep replicas off the same node | **Pod anti-affinity** (careful: hard rules can leave Pods Pending) |
| Spread replicas evenly over nodes or zones | **Topology spread constraints** |
| Reserve nodes for one workload | **Taints + tolerations + nodeSelector/affinity** |
| Stop new Pods landing on a node for maintenance | `kubectl cordon` (a taint) |
| Evict Pods from a node | `NoExecute` taint, or `kubectl drain` |
| Make sure a Pod gets capacity | **Requests** |
| Cap what a Pod can use | **Limits** |
| Decide who is evicted first, or who preempts whom | **QoS class**, **PriorityClass** |
| Pin to one named node | `nodeName` (avoid) |

## Hard vs soft

| | Hard (filter) | Soft (score) |
|---|---|---|
| Node affinity | `required...` | `preferred...` with `weight` |
| Pod affinity | `required...` | `preferred...` |
| Topology spread | `DoNotSchedule` | `ScheduleAnyway` |
| Taints | `NoSchedule`, `NoExecute` | `PreferNoSchedule` |
| If nothing fits | Pod **Pending** | Pod placed somewhere else |

**Rule of thumb:** use hard rules for correctness (a GPU is mandatory), soft rules for optimization (prefer cheaper nodes). Too many hard rules cause `Pending` Pods.

## Debug: "my Pod is Pending"

```bash
kubectl describe pod <name> | grep -A5 Events
```

| Event message | Meaning | Look at |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Requests do not fit | [Requests](resource-requests-limits.md) |
| `didn't match Pod's node affinity/selector` | No node has the labels | [Node selector](node-selector.md), [affinity](node-affinity.md) |
| `had untolerated taint` | Nodes are tainted | [Taints](taints-tolerations.md) |
| `didn't match pod anti-affinity rules` | Anti-affinity cannot be met | [Pod affinity](pod-affinity.md) |
| `didn't satisfy existing pods anti-affinity rules` | Another Pod's rules block it | [Pod affinity](pod-affinity.md) |
| `unbound immediate PersistentVolumeClaims` | Storage problem | [Storage](../06-storage/) |

Each of these messages starts with `0/N nodes are available:` followed by a breakdown per reason.

## Recommended study order

| # | Topic | Why now |
|---|---|---|
| 1 | [Labels and Selectors](labels-selectors.md) | Every rule here uses labels |
| 2 | [Node Selector](node-selector.md) | The simplest rule |
| 3 | [Node Affinity](node-affinity.md) | The expressive version |
| 4 | [Pod Affinity](pod-affinity.md) | Rules about other Pods, plus topology spread |
| 5 | [Taints and Tolerations](taints-tolerations.md) | The node's side of the conversation |
| 6 | [Resource Requests and Limits](resource-requests-limits.md) | Sizing, QoS and priority |

## Labs

- Needs a **multi-node** cluster: `minikube start -p sched --nodes 3` or `kind create cluster --name sched --config labs/07-scheduling/kind-cluster.yaml`.
- [labs/07-scheduling](../labs/07-scheduling/): setup, plus a combined "mini platform policy" scenario
- Each note has its own lab with numbered steps.
- Control-plane nodes are tainted in kind and kubeadm clusters, and open in minikube. The notes pick two **worker** nodes to avoid the difference.

## Module quiz

1. What does the scheduler do in its two phases?
2. What is the difference between a label and an annotation?
3. What happens to a Pod whose `nodeSelector` matches no node?
4. What does "IgnoredDuringExecution" mean?
5. How do multiple `nodeSelectorTerms` combine, and how do multiple `matchExpressions` inside one term combine?
6. What does `topologyKey: topology.kubernetes.io/zone` mean in an anti-affinity rule?
7. Five replicas, required anti-affinity per node, three nodes. What happens?
8. What is `maxSkew`, and why choose topology spread over anti-affinity?
9. Does a toleration make a Pod go to a tainted node?
10. Which taint effect evicts running Pods?
11. How do you reserve nodes for one workload?
12. Why are Pods on a dead node replaced after about five minutes?
13. What do requests and limits each do?
14. What happens when a container exceeds its CPU limit? Its memory limit?
15. What QoS class has no requests or limits, and what does it mean?
16. What does a PriorityClass add?

<details>
<summary>Answers</summary>

1. Filter out nodes that cannot run the Pod, then score the rest and bind to the best
2. Labels identify and select. Annotations hold larger non-identifying metadata and cannot be selected on
3. It stays `Pending`
4. Rules apply only when scheduling. If labels change later, running Pods are not evicted
5. Terms are ORed. Expressions inside one term are ANDed
6. Each Pod must be in a different availability zone from the matching Pods
7. Three Pods run and two stay Pending
8. The allowed difference in Pod count between the fullest and emptiest domain. Topology spread places extra replicas evenly instead of leaving them Pending
9. No. It only allows it. Use a nodeSelector or affinity to attract
10. `NoExecute`
11. Taint the nodes, label them, and give the workload a toleration plus a nodeSelector or affinity
12. They tolerate the `not-ready` and `unreachable` taints for 300 seconds by default
13. Requests: the reserved minimum used for scheduling and CPU shares. Limits: the enforced ceiling
14. CPU: throttled. Memory: killed (`OOMKilled`, exit code 137)
15. BestEffort: evicted first under node pressure
16. An importance value used for scheduling order and preemption of lower-priority Pods

</details>

## Self-test before moving on

1. Make a Pod `Pending` in four different ways and read each reason from the Events.
2. Reserve one node for a workload and prove nothing else lands there.
3. Spread six replicas evenly over three nodes, then break it with required anti-affinity.
4. Create Guaranteed, Burstable and BestEffort Pods and read their classes.
5. Take the module quiz without looking at the answers.

## How to study this module

1. Read each note in order.
2. Type the YAML yourself, do not copy-paste.
3. Run the lab, then break it on purpose.
4. When you have studied and practiced a note, change its `Status` line to `✅ Studied`.

Next: [Module 08: Health and Scaling](../08-health-and-scaling/)