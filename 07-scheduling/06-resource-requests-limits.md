# Resource Requests and Limits

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-scheduling/manifests/`](../labs/07-scheduling/manifests/) · Cluster setup: [`labs/07-scheduling`](../labs/07-scheduling/)

[← Taints and Tolerations](taints-tolerations.md) | [README](README.md)

---

## Idea

Every container can declare how much CPU and memory it **needs** and how much it may **use**:

```yaml
resources:
  requests:            # what I need: used by the SCHEDULER, and as a guaranteed minimum
    cpu: 100m
    memory: 128Mi
  limits:              # the ceiling: enforced at RUNTIME by the kubelet and the kernel
    cpu: 500m
    memory: 256Mi
```

| | **Requests** | **Limits** |
|---|---|---|
| Question answered | "How much should be **reserved** for me?" | "How much may I **never exceed**?" |
| Used by | The **scheduler** (does it fit?) and CPU sharing | The **kubelet and kernel** (throttle or kill) |
| If missing | Treated as 0 (or the LimitRange default) | No ceiling (or the LimitRange default) |

## Units

| Resource | Notation | Meaning |
|---|---|---|
| **CPU** | `1`, `0.5`, `500m` | `1` = one core (a vCPU). `500m` = 500 millicores = half a core. Smallest is `1m` |
| **Memory** | `128Mi`, `1Gi` | **Mebibytes / gibibytes** (powers of 1024) |
| Memory (decimal) | `128M`, `1G` | Megabytes (powers of 1000). `128Mi` ≠ `128M` |
| **Ephemeral storage** | `1Gi` | Container writable layer, `emptyDir`, logs |

Writing `memory: 128m` means 0.128 bytes, a common typo. Use `Mi` and `Gi`.

## How requests are used: scheduling

The scheduler only places a Pod on a node if the **sum of requests** of the Pods there plus the new one fits in the node's **allocatable** capacity. Actual usage does not matter.

```
Node allocatable: 2000m CPU
Pod A request 800m + Pod B request 800m = 1600m used (even if they idle)
New Pod request 500m → 2100m > 2000m → does NOT fit on this node
```

```bash
kubectl describe node <node> | grep -A8 "Allocated resources"
```

If no node fits, the Pod is `Pending` with `Insufficient cpu` or `Insufficient memory`. This is also what the **Cluster Autoscaler / Karpenter** react to: Pending Pods' requests decide how many nodes get added.

## How limits are enforced: runtime

| Resource | Exceeding the limit | Result |
|---|---|---|
| **CPU** | The kernel **throttles** the container (CFS quota) | Slower, **never killed** |
| **Memory** | The kernel's OOM killer kills the container | **`OOMKilled`**, exit code **137**, restarted per `restartPolicy` |
| **Ephemeral storage** | The kubelet evicts the Pod | Pod `Evicted` |

And requests also matter at runtime:

- **CPU requests** set the **share** each container gets when the node is busy (proportional split).
- **Memory requests** decide who is **evicted first** under node memory pressure.

```
CPU:     limit exceeded  →  slowed down (throttled)
Memory:  limit exceeded  →  killed (OOMKilled)
```

## QoS classes

Kubernetes assigns each Pod a **Quality of Service** class from its requests and limits:

| Class | Rule | Under node pressure |
|---|---|---|
| **Guaranteed** | **Every** container has CPU and memory requests **and** limits, **requests = limits** | Evicted **last** |
| **Burstable** | At least one request or limit set, but not Guaranteed | Middle (those using most above their requests go first) |
| **BestEffort** | **No** requests or limits at all | Evicted **first** |

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}{"\n"}'
```

## Choosing values

1. **Measure first**: `kubectl top pod --containers` (needs metrics-server), Prometheus, or VPA in recommend mode.
2. **Requests**: near typical usage (somewhere between the median and a high percentile). Too low causes noisy neighbors and early eviction. Too high wastes money and leaves Pods `Pending`.
3. **Memory limit**: equal to the memory request, or slightly above. Memory cannot be "borrowed back" like CPU, so reaching the limit kills the container.
4. **CPU limit**: debated. Many teams set CPU **requests** but omit CPU **limits** to avoid throttling, while keeping memory limits. Setting `requests = limits` gives the Guaranteed class.
5. **Language runtimes**: make the runtime stay under the limit. For Node.js, set the heap explicitly, for example about 75% of the memory limit:

```yaml
env:
  - name: NODE_OPTIONS
    value: "--max-old-space-size=384"      # for a 512Mi memory limit
resources:
  requests: { cpu: 200m, memory: 512Mi }
  limits:   { memory: 512Mi }
```

Autoscaling builds on this: the **HPA** computes utilization as usage divided by the **request** (see [Module 08](../08-health-and-scaling/)), so a Deployment without CPU requests cannot be autoscaled on CPU.

## LimitRange: defaults and bounds per namespace

A **LimitRange** fills in missing values and rejects extreme ones when Pods are **created** (it does not change existing Pods).

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: quota-demo
spec:
  limits:
    - type: Container
      defaultRequest:         # used when a container sets no request
        cpu: 100m
        memory: 64Mi
      default:                # used when a container sets no limit
        cpu: 200m
        memory: 128Mi
      max:
        memory: 256Mi         # reject anything above this
```

## ResourceQuota: a cap for the whole namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: quota-demo
spec:
  hard:
    pods: "3"
    requests.cpu: 500m
    requests.memory: 512Mi
    limits.cpu: "1"
    limits.memory: 1Gi
```

- Creating something that would exceed the quota is **rejected** (`exceeded quota`).
- When a quota covers CPU or memory, every new Pod **must** state them. A LimitRange with defaults makes that automatic.
- Quotas can also limit object counts (`count/deployments.apps`), PVCs and storage (`requests.storage`).

## Priority and preemption: when the cluster is full

A **PriorityClass** gives Pods an importance number. If a high-priority Pod cannot fit, the scheduler can **preempt** (evict) lower-priority Pods to make room.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
---
# in the Pod
spec:
  priorityClassName: high-priority
```

| Setting | Meaning |
|---|---|
| `value` | Higher number = more important (default Pod priority is 0) |
| `preemptionPolicy: Never` | Queue ahead of lower priorities, but never evict others |
| built-ins | `system-cluster-critical`, `system-node-critical` (for Kubernetes' own components) |

Use it to protect critical services and let batch work yield.

## Lab

```bash
cd labs/07-scheduling/manifests
```

**1. Requests decide placement**

```bash
kubectl apply -f resources-demo.yaml
NODE=$(kubectl get pod resources-demo -o jsonpath='{.spec.nodeName}'); echo $NODE
kubectl describe node $NODE | grep -A8 "Allocated resources"      # includes its 100m CPU / 64Mi request
kubectl apply -f big-request.yaml                                  # requests 100 CPUs
kubectl get pod big-request                                        # Pending
kubectl describe pod big-request | grep -A3 Events                 # Insufficient cpu
```

**2. QoS classes**

```bash
kubectl apply -f qos-demo.yaml
kubectl get pod qos-guaranteed qos-burstable qos-besteffort -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

Expected: Guaranteed, Burstable, BestEffort.

**3. Memory limit: OOMKilled**

```bash
kubectl apply -f memory-hog.yaml                                   # tries to use 150M under a 100Mi limit
kubectl get pod memory-hog -w                                      # OOMKilled, then CrashLoopBackOff
kubectl describe pod memory-hog | grep -A6 "Last State"            # Reason: OOMKilled, Exit Code: 137
```

**4. CPU limit: throttled, not killed**

```bash
kubectl apply -f cpu-throttle.yaml                                 # busy loop with a 100m limit
sleep 30
kubectl exec cpu-throttle -- cat /sys/fs/cgroup/cpu.stat           # cgroup v2: look at nr_throttled
# cgroup v1 nodes: kubectl exec cpu-throttle -- cat /sys/fs/cgroup/cpu/cpu.stat
kubectl get pod cpu-throttle                                       # still Running
# with metrics-server (minikube addons enable metrics-server):
kubectl top pod cpu-throttle                                       # close to 100m, never above
```

`nr_throttled` rising while the Pod keeps running shows the difference from memory.

**5. LimitRange and ResourceQuota**

```bash
kubectl apply -f limitrange-quota.yaml
kubectl run nores --image=nginx:1.27 -n quota-demo
kubectl get pod nores -n quota-demo -o jsonpath='{.spec.containers[0].resources}{"\n"}'   # defaults were injected
kubectl describe quota compute-quota -n quota-demo

kubectl create deployment web -n quota-demo --image=nginx:1.27 --replicas=5
kubectl get deploy web -n quota-demo                                # fewer than 5 ready: the pod quota is 3
kubectl describe rs -n quota-demo | grep -i "exceeded quota"

kubectl apply -f limitrange-violation.yaml                          # rejected: above the 256Mi max
```

**6. Priority**

```bash
kubectl apply -f priorityclass.yaml
kubectl get priorityclass
kubectl get pod priority-demo -o jsonpath='{.spec.priority}{"\n"}'  # 1000000
```

**Cleanup**

```bash
kubectl delete -f resources-demo.yaml -f big-request.yaml -f qos-demo.yaml -f memory-hog.yaml -f cpu-throttle.yaml --ignore-not-found
kubectl delete -f priorityclass.yaml --ignore-not-found
kubectl delete namespace quota-demo
```

## Gotchas

- **No requests means BestEffort**, the first to be evicted, and the scheduler thinks the Pod uses nothing.
- **`memory: 128m` is a typo** for `128Mi`.
- **OOMKilled = memory limit.** The fix is a higher limit or fixing the leak, not a higher CPU.
- **CPU throttling** can cause latency even when average CPU looks low (short bursts hit the quota).
- The **scheduler uses requests, not live usage.** A cluster can be "full" with idle Pods.
- A **LimitRange affects only new Pods**, and a ResourceQuota blocks creation without failing existing Pods.
- Requests and limits are per **container**. A Pod's total is the sum of its containers.
- Without CPU requests, an **HPA on CPU utilization** has nothing to divide by.

## Check yourself

1. What is the difference between a request and a limit?
2. What happens when a container exceeds its CPU limit? Its memory limit?
3. What QoS class is a Pod with no requests or limits, and what does that mean under memory pressure?
4. How does the scheduler decide whether a Pod fits on a node?
5. What is `OOMKilled`, and what exit code does it show?
6. What do LimitRange and ResourceQuota each do?
7. What does a PriorityClass add?

<details>
<summary>Answers</summary>

1. A request is the reserved minimum used for scheduling and CPU sharing. A limit is the enforced ceiling
2. CPU: throttled, never killed. Memory: the container is killed (`OOMKilled`, exit code 137)
3. BestEffort: evicted first when the node is short of resources
4. It sums the requests of the Pods on the node plus the new Pod and checks them against the node's allocatable capacity
5. The kernel killed the container for exceeding its memory limit. Exit code 137
6. LimitRange sets per-container defaults and min/max in a namespace. ResourceQuota caps total resources and object counts for the namespace
7. An importance value: higher-priority Pods are scheduled first and can preempt lower-priority ones when the cluster is full

</details>

---

[← Taints and Tolerations](taints-tolerations.md) | [README](README.md)