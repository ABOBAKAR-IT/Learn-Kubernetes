# Vertical Pod Autoscaler (VPA)

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-autoscaling/manifests/`](../labs/07-autoscaling/manifests/) · Lab: [`labs/07-autoscaling`](../labs/07-autoscaling/)

[← Horizontal Pod Autoscaler](horizontal-pod-autoscaler.md) | [README](README.md)

---

## Idea

The **Vertical Pod Autoscaler** adjusts the **CPU and memory requests** of your Pods, making each Pod **bigger or smaller** instead of adding more of them. It watches how much a workload really uses and recommends (or applies) better numbers.

```
HPA:  more Pods of the same size         ──►  [Pod][Pod][Pod][Pod]
VPA:  the same number of bigger Pods     ──►  [ Pod ][ Pod ]
```

It solves the hardest part of [requests and limits](../07-scheduling/resource-requests-limits.md): **picking the right numbers**. Guess too low and Pods get evicted or throttled. Guess too high and you pay for idle capacity.

> VPA is **not built into every cluster**. It is a separate add-on (from the Kubernetes autoscaler project) that you install. Some managed platforms offer it as an option, and on others you install it yourself.

## Components

| Component | Job |
|---|---|
| **Recommender** | Watches usage (from metrics-server and optionally Prometheus) and computes recommended requests |
| **Updater** | Applies recommendations to **running** Pods, by evicting them or by resizing them in place |
| **Admission controller** | A webhook that sets the recommended requests on **new** Pods as they are created |

```
metrics ─► Recommender ─► VPA object (status.recommendation)
                              │
              ┌───────────────┴────────────────┐
         Admission controller             Updater
         (sets requests on new Pods)      (evicts or resizes running Pods)
```

## YAML explained

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
spec:
  targetRef:                       # WHAT to manage
    apiVersion: apps/v1
    kind: Deployment
    name: hamster
  updatePolicy:
    updateMode: "Off"              # quote it: a bare Off is YAML for false
  resourcePolicy:
    containerPolicies:
      - containerName: "*"         # all containers, or name one
        minAllowed:
          cpu: 25m
          memory: 16Mi
        maxAllowed:
          cpu: "1"
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits     # or RequestsOnly
```

| Field | Meaning |
|---|---|
| `targetRef` | The workload whose Pods are managed |
| `updatePolicy.updateMode` | How recommendations are applied (table below) |
| `minAllowed` / `maxAllowed` | Bounds for recommendations (always set a `maxAllowed` below your biggest node) |
| `controlledResources` | Which resources VPA manages |
| `controlledValues` | `RequestsAndLimits` keeps the limit proportional to the request. `RequestsOnly` changes only requests |

## Update modes

| Mode | What VPA does | Disruption |
|---|---|---|
| **`Off`** | Only **calculates recommendations**. Applies nothing | None |
| **`Initial`** | Sets requests only when Pods are **created**. Never touches running Pods | None to running Pods |
| **`Recreate`** | **Evicts** Pods whose requests differ much from the recommendation, and new Pods get the new values | Restarts |
| **`InPlaceOrRecreate`** | Tries to **resize the running Pod in place**. Falls back to eviction if it cannot | Low |
| **`InPlace`** | Resizes in place and **never evicts**. Retries while the resize is deferred (newer VPA versions) | Lowest |
| `Auto` | **Deprecated** alias of `Recreate` | Restarts |

The in-place modes rely on Kubernetes' **in-place Pod resize** feature, so they need a cluster version that supports it, and some VPA versions need a feature gate. Check the VPA documentation for your versions.

**A safe way to start: run in `Off` mode**, read the recommendations for a few days, and apply them yourself (or switch to `Initial` or `Recreate` later).

## Reading recommendations

```bash
kubectl describe vpa hamster-vpa
```

```
Recommendation:
  Container Recommendations:
    Container Name:  hamster
    Lower Bound:     Cpu: 25m   Memory: 25Mi
    Target:          Cpu: 142m  Memory: 48Mi        ← what VPA would set
    Uncapped Target: Cpu: 142m  Memory: 48Mi        ← before minAllowed/maxAllowed
    Upper Bound:     Cpu: 960m  Memory: 500Mi
```

| Value | Meaning |
|---|---|
| **Target** | The recommended request |
| **Lower Bound** | The least you should ask for |
| **Upper Bound** | Above this is probably wasted |
| **Uncapped Target** | The recommendation ignoring your min/max policy |

The recommender needs **some history** before it is meaningful. Early values are low-confidence and wide.

## In-place Pod resize (the Kubernetes feature behind it)

Newer Kubernetes versions let you change a running Pod's CPU and memory **without recreating it**, through the `resize` subresource, and each resource says whether a container restart is needed:

```yaml
containers:
  - name: app
    image: nginx:1.27
    resizePolicy:
      - resourceName: cpu
        restartPolicy: NotRequired         # change CPU live
      - resourceName: memory
        restartPolicy: RestartContainer    # restart the container to change memory
```

```bash
kubectl patch pod resize-demo --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"200m"},"limits":{"cpu":"200m"}}}]}}'
```

Limits: the Pod's QoS class cannot change, and the node must have the room.

## VPA vs HPA

| | **HPA** | **VPA** |
|---|---|---|
| Changes | Number of replicas | Size of each Pod |
| Reacts | In seconds to minutes | Slowly (needs history) |
| Disruption | None (adds or removes Pods) | Restart (or in-place resize) |
| Works for | Stateless, replicable apps | Any app, including a single replica that cannot scale out |
| Needs | metrics-server, CPU requests | VPA add-on installed |

**Do not let both control CPU or memory on the same workload.** Both would react to the same signal and fight. Valid combinations: VPA in `Off` mode for recommendations alongside an HPA, or an HPA on a custom metric (such as requests per second) with VPA managing CPU and memory.

## When VPA fits

| Good fit | Poor fit |
|---|---|
| Right-sizing: finding sensible requests for many services | Workloads that cannot tolerate restarts (unless using in-place) |
| Workloads that cannot scale out (single instance, some databases) | Latency-critical single replicas under `Recreate` |
| Batch jobs with unpredictable size | A workload already driven by an HPA on CPU |
| Reducing cost from over-provisioned requests | Apps with usage spikes much higher than their normal use |

Related tools: **Goldilocks** (runs VPA in recommendation mode and shows a dashboard), and cost tools that suggest requests.

## Lab

Needs metrics-server and the VPA add-on. Install VPA from the project repository (check its README for the current method):

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
kubectl get pods -n kube-system | grep vpa            # recommender, updater, admission-controller
cd -
cd labs/07-autoscaling/manifests
```

**1. Recommendation-only mode**

```bash
kubectl apply -f vpa-hamster.yaml                     # 2 replicas, tiny requests (20m CPU), a mostly idle busy-loop
kubectl get pods -l app=hamster
# wait about 5 minutes for data, then:
kubectl describe vpa hamster-vpa | tail -20           # Lower Bound / Target / Upper Bound
kubectl get vpa hamster-vpa -o jsonpath='{.status.recommendation.containerRecommendations[0].target}{"\n"}'
kubectl get pods -l app=hamster -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].resources.requests}{"\n"}{end}'
```

Compare the **Target** with the Pods' real requests (`20m`). `Off` mode changed nothing.

**2. Let VPA apply it (`Recreate`)**

```bash
kubectl patch vpa hamster-vpa --type=merge -p '{"spec":{"updatePolicy":{"updateMode":"Recreate"}}}'
kubectl get pods -l app=hamster -w                    # Pods are evicted one at a time and recreated
kubectl get pods -l app=hamster -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].resources.requests}{"\n"}{end}'
kubectl get pod -l app=hamster -o jsonpath='{.items[0].metadata.annotations}{"\n"}'     # an annotation records the VPA change
```

The new Pods carry the recommended requests. The updater needs **at least two replicas** (the default) so a restart does not take the whole service down.

**3. In-place resize on a single Pod** (needs a Kubernetes version with the resize subresource)

```bash
kubectl apply -f resize-demo.yaml
kubectl get pod resize-demo -o jsonpath='{.status.containerStatuses[0].restartCount}{"\n"}'       # 0
kubectl patch pod resize-demo --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"200m"},"limits":{"cpu":"200m"}}}]}}'
kubectl get pod resize-demo -o jsonpath='{.spec.containers[0].resources}{"\n"}'                  # 200m
kubectl get pod resize-demo -o jsonpath='{.status.containerStatuses[0].restartCount}{"\n"}'      # still 0
```

**Cleanup**

```bash
kubectl delete -f vpa-hamster.yaml -f resize-demo.yaml
cd autoscaler/vertical-pod-autoscaler && ./hack/vpa-down.sh    # remove VPA when finished
```

## Gotchas

- **`updateMode: Off` must be quoted** (`"Off"`). An unquoted `Off` is parsed as the boolean `false`.
- `Recreate` **restarts Pods**. Use multiple replicas and PodDisruptionBudgets, or an in-place mode.
- **VPA and HPA on the same CPU or memory metric conflict.**
- Recommendations need **time and real load**. A new or idle workload gets low-confidence values.
- Always set **`maxAllowed`** so VPA cannot ask for more than any node offers (that would leave Pods `Pending`).
- VPA changes requests, so cost and node packing change too. Check the capacity.
- VPA adjusts **requests** (and proportionally limits). It does not fix memory leaks.

## Check yourself

1. What does VPA change that HPA does not?
2. Name the three VPA components and what each does.
3. What is the difference between `Off`, `Initial` and `Recreate`?
4. What does `InPlaceOrRecreate` add?
5. Why must you not run an HPA and a VPA on the same CPU metric?
6. Why is `Off` mode a good first step?

<details>
<summary>Answers</summary>

1. The size (CPU and memory requests) of each Pod, instead of the number of Pods
2. Recommender (computes recommendations), Updater (evicts or resizes running Pods), Admission controller (sets requests on new Pods)
3. `Off`: recommendations only. `Initial`: applies requests only when Pods are created. `Recreate`: also evicts running Pods to apply new requests
4. It resizes the running Pod in place when possible and only evicts as a fallback
5. They react to the same signal and fight each other
6. It shows recommendations with no risk, so you can review them before applying anything

</details>

---

[← Horizontal Pod Autoscaler](horizontal-pod-autoscaler.md) | [README](README.md)