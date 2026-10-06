# 🗂️ Kubernetes Namespaces & Resource Management – Beginner Friendly Guide

Learn how to **organize a cluster with Namespaces** and **control CPU and memory** with requests, limits, ResourceQuota and LimitRange, using a simple hands-on example.

---

## 📚 Table of Contents

1. [What is a Namespace?](#-what-is-a-namespace)
2. [Why do we need Resource Management?](#-why-do-we-need-resource-management)
3. [Project Structure](#-project-structure)
4. [Prerequisites](#-prerequisites)
5. [Step-by-Step Walkthrough](#-step-by-step-walkthrough)
6. [Understanding Each File](#-understanding-each-file)
7. [Requests vs Limits](#-requests-vs-limits)
8. [CPU and Memory Units](#-cpu-and-memory-units)
9. [What happens when a limit is crossed?](#-what-happens-when-a-limit-is-crossed)
10. [QoS Classes](#-qos-classes)
11. [ResourceQuota – limit a whole namespace](#-resourcequota--limit-a-whole-namespace)
12. [LimitRange – default values per container](#-limitrange--default-values-per-container)
13. [Useful Namespace Commands](#-useful-namespace-commands)
14. [Common Mistakes](#-common-mistakes)
15. [Cheat Sheet](#-cheat-sheet)
16. [Cleanup](#-cleanup)

---

## 🏠 What is a Namespace?

A **Namespace** is a virtual cluster inside your real cluster. It groups resources (Pods, Services, Deployments, ConfigMaps, Secrets...) and gives them a **separate name scope**.

Think of a cluster as an **apartment building** and each namespace as an **apartment**:

```
Kubernetes Cluster
├── Namespace: dev-ns      → dev team's Pods, Services
├── Namespace: test-ns     → QA team's Pods, Services
└── Namespace: prod-ns     → production workloads
```

### Why use Namespaces?

| Benefit | Explanation |
|---|---|
| **Isolation** | Teams or environments (dev/test/prod) stay separate |
| **Same names allowed** | `my-app` can exist in both `dev-ns` and `prod-ns` |
| **Access control** | RBAC can give a user access to only one namespace |
| **Resource control** | Apply quotas per namespace |
| **Easy cleanup** | Deleting a namespace deletes everything inside it |

### Default namespaces

| Namespace | Purpose |
|---|---|
| `default` | Where objects go if you don't specify a namespace |
| `kube-system` | Kubernetes system components (DNS, kube-proxy, etc.) |
| `kube-public` | Publicly readable data (rarely used) |
| `kube-node-lease` | Node heartbeat objects |

> ⚠️ Namespaces provide **logical** separation, not strong security isolation by themselves. For network isolation you need NetworkPolicies.

> ℹ️ Some objects are **cluster-wide** and do **not** belong to any namespace (for example Nodes, PersistentVolumes, and Namespaces themselves). Check with `kubectl api-resources --namespaced=false`.

---

## ⚙️ Why do we need Resource Management?

Without limits, one buggy Pod can eat all the memory or CPU on a node and hurt every other Pod.

Kubernetes gives you **four tools**:

| Tool | Level | What it does |
|---|---|---|
| **Requests** | Container | Minimum guaranteed resources (used for scheduling) |
| **Limits** | Container | Maximum a container may use |
| **ResourceQuota** | Namespace | Total cap for the whole namespace |
| **LimitRange** | Namespace | Default / min / max values for containers |

```
ResourceQuota  ─►  "dev-ns may use at most 1 CPU and 1Gi memory in total"
LimitRange     ─►  "each container gets 100m CPU by default, max 500m"
Requests/Limit ─►  "this container needs 100m and may use up to 200m"
```

---

## 📁 Project Structure

```
namespaces-resource-management/
├── namespace.yaml        # Creates dev-ns
├── pod-limit.yaml        # Pod with requests and limits
├── resource-quota.yaml   # (bonus) namespace-wide cap
├── limit-range.yaml      # (bonus) default values
└── README.md
```

---

## ✅ Prerequisites

- A running Kubernetes cluster (Minikube, Kind, Docker Desktop or cloud)
- `kubectl` installed and configured
- *(Optional)* `metrics-server` installed to use `kubectl top`

```bash
kubectl cluster-info
kubectl get nodes
```

---

## 🚀 Step-by-Step Walkthrough

### Step 1 – Create the Namespace

```bash
kubectl apply -f namespace.yaml
kubectl get namespaces
```

### Step 2 – Create the Pod with requests and limits

```bash
kubectl apply -f pod-limit.yaml
```

### Step 3 – Check the Pod

```bash
kubectl get pods -n dev-ns
```

> 💡 Use `-n dev-ns` every time. Without it, kubectl looks in the `default` namespace and says *"No resources found"*.

### Step 4 – Inspect resources in detail

```bash
kubectl describe pod limited-pod -n dev-ns
```

Look for this section:

```
Limits:
  cpu:     200m
  memory:  256Mi
Requests:
  cpu:     100m
  memory:  128Mi
QoS Class: Burstable
```

### Step 5 – (Optional) See live usage

```bash
kubectl top pod limited-pod -n dev-ns
```

This needs `metrics-server`. On Minikube: `minikube addons enable metrics-server`.

---

## 🔍 Understanding Each File

### 1️⃣ `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev-ns
```

| Field | Meaning |
|---|---|
| `kind: Namespace` | The object type |
| `metadata.name` | Name of the namespace (lowercase letters, numbers, `-`) |

Quick alternative without YAML:

```bash
kubectl create namespace dev-ns
```

---

### 2️⃣ `pod-limit.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
  namespace: dev-ns
spec:
  containers:
    - name: busybox-container
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "256Mi"
          cpu: "200m"
```

| Field | Meaning |
|---|---|
| `metadata.namespace: dev-ns` | Puts the Pod in `dev-ns` (the namespace must already exist) |
| `image: busybox` | A tiny Linux image. ⚠️ The image name must be real, e.g. `busybox`, not `busybox-container` |
| `command` | Keeps the container running for 1 hour |
| `requests` | Minimum reserved for scheduling |
| `limits` | Maximum allowed at runtime |

---

## ⚖️ Requests vs Limits

| | Request | Limit |
|---|---|---|
| Meaning | "I **need** at least this much" | "I must **never exceed** this" |
| Used by | **Scheduler** (choosing a node) | **Kubelet / container runtime** (enforcing at runtime) |
| Guaranteed? | Yes, reserved for the container | It's a ceiling, not a reservation |

**Analogy: a restaurant table**
- *Request* = the table you reserve (4 seats guaranteed)
- *Limit* = the maximum seats you may use if the restaurant is empty (6 seats)

Example for our Pod:

```
Node has 1000m CPU free
limited-pod requests 100m   → scheduler: "fits, place it here"
limited-pod can burst to 200m if the node has spare CPU
```

If **no node** has enough free resources to satisfy the **requests**, the Pod stays in `Pending`.

---

## 📏 CPU and Memory Units

### CPU

| Value | Meaning |
|---|---|
| `1` | 1 full CPU core |
| `500m` | 0.5 core (m = millicores, 1000m = 1 core) |
| `100m` | 0.1 core = **10% of one core** |
| `200m` | 0.2 core = **20% of one core** |

> Note: `100m` is 10% of **one core**, not 10% of the whole machine. On a 4-core node it is only 2.5% of the node.

### Memory

| Unit | Meaning | Bytes |
|---|---|---|
| `Mi` | Mebibyte | 1024 × 1024 |
| `Gi` | Gibibyte | 1024 Mi |
| `M` | Megabyte | 1,000,000 |
| `G` | Gigabyte | 1,000,000,000 |

> ⚠️ `256Mi` and `256M` are **not** identical. Prefer `Mi` / `Gi`. Also watch out: `m` in memory means *milli-bytes*, which is almost never what you want.

---

## 💥 What happens when a limit is crossed?

| Resource | Type | If the container exceeds the limit |
|---|---|---|
| **CPU** | Compressible | The container is **throttled** (slowed down), not killed |
| **Memory** | Incompressible | The container is **OOMKilled** (Out Of Memory) and restarted |

Check for an OOM kill:

```bash
kubectl describe pod limited-pod -n dev-ns
# Last State:  Terminated
#   Reason:    OOMKilled
```

---

## 🏷️ QoS Classes

Kubernetes assigns every Pod a **Quality of Service** class based on its requests and limits. This decides who gets evicted first when a node runs out of resources.

| QoS Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | Every container has requests **equal to** limits for both CPU and memory | Last to be evicted |
| **Burstable** | At least one request or limit set, but not Guaranteed | Middle |
| **BestEffort** | No requests or limits at all | First to be evicted |

Our `limited-pod` has requests lower than limits, so it is **Burstable**.

Check any Pod's class:

```bash
kubectl get pod limited-pod -n dev-ns -o jsonpath='{.status.qosClass}'
```

---

## 🧮 ResourceQuota – limit a whole namespace

A **ResourceQuota** caps the **total** resources and object counts in a namespace.

`resource-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev-ns
spec:
  hard:
    pods: "5"
    requests.cpu: "500m"
    requests.memory: "512Mi"
    limits.cpu: "1"
    limits.memory: "1Gi"
```

Apply and inspect:

```bash
kubectl apply -f resource-quota.yaml
kubectl get resourcequota -n dev-ns
kubectl describe resourcequota dev-quota -n dev-ns
```

Sample output:

```
Resource         Used   Hard
--------         ----   ----
limits.cpu       200m   1
limits.memory    256Mi  1Gi
pods             1      5
requests.cpu     100m   500m
requests.memory  128Mi  512Mi
```

### Important rules

- Once a quota covers `requests.cpu` / `limits.memory` etc., **every new Pod in that namespace must specify those values**, otherwise it is rejected.
- Creating a Pod that would exceed the quota fails immediately with an error like:

```
Error from server (Forbidden): pods "big-pod" is forbidden: exceeded quota: dev-quota
```

- Quotas only affect objects created **after** the quota exists.
- You can also cap object counts: `services`, `configmaps`, `secrets`, `persistentvolumeclaims`, etc.

---

## 📐 LimitRange – default values per container

A **LimitRange** sets **default**, **minimum** and **maximum** values per container in a namespace. It saves you from writing resources in every Pod and stops people from asking for too much.

`limit-range.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev-ns
spec:
  limits:
    - type: Container
      defaultRequest:      # used if a container has no request
        cpu: "100m"
        memory: "128Mi"
      default:             # used if a container has no limit
        cpu: "200m"
        memory: "256Mi"
      min:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "500m"
        memory: "512Mi"
```

```bash
kubectl apply -f limit-range.yaml
kubectl describe limitrange dev-limits -n dev-ns
```

### Quota vs LimitRange

| | ResourceQuota | LimitRange |
|---|---|---|
| Scope | Whole namespace total | Each container / Pod |
| Purpose | "Don't use more than X in total" | "Each container gets defaults and stays within min/max" |
| Sets defaults? | ❌ | ✅ |
| Limits object count? | ✅ | ❌ |

> ✅ Best practice: use **both** together. LimitRange fills in defaults, so Pods that forget resources still satisfy the quota.

---

## 🧭 Useful Namespace Commands

```bash
# List all namespaces
kubectl get ns

# Create / delete
kubectl create namespace test-ns
kubectl delete namespace test-ns

# Work inside a namespace
kubectl get pods -n dev-ns
kubectl get all -n dev-ns

# Look across ALL namespaces
kubectl get pods -A

# Set a default namespace for your current context (so you can skip -n)
kubectl config set-context --current --namespace=dev-ns

# Check which namespace you're currently using
kubectl config view --minify | grep namespace:
```

### Talking to a Service in another namespace

Services get a DNS name that includes the namespace:

```
<service-name>.<namespace>.svc.cluster.local
```

Example: a Pod in `dev-ns` can reach `db` in `prod-ns` using `db.prod-ns.svc.cluster.local`. Using only `db` resolves within the **same** namespace.

---

## 🐞 Common Mistakes

| Problem | Cause | Fix |
|---|---|---|
| `Error: namespaces "dev-ns" not found` | Applied the Pod before the namespace | Run `kubectl apply -f namespace.yaml` first |
| Pod is `ImagePullBackOff` / `ErrImagePull` | Wrong image name (e.g. `busybox-container`) | Use a real image like `busybox` |
| `No resources found in default namespace` | Forgot `-n dev-ns` | Add `-n dev-ns` or set the context namespace |
| Pod stuck in `Pending` | Requests are bigger than any node can provide | Lower requests or add nodes. Check `kubectl describe pod` → Events |
| `OOMKilled` | Memory usage crossed the limit | Raise the memory limit or fix the memory leak |
| `exceeded quota` error | Namespace total would exceed ResourceQuota | Free resources, delete Pods, or raise the quota |
| `must specify limits.cpu` error | Quota exists but Pod has no resources | Add resources to the Pod or create a LimitRange with defaults |
| `kubectl top` fails: *Metrics API not available* | metrics-server not installed | Install/enable metrics-server |
| Limit lower than request | Invalid configuration | Always keep `limit >= request` |
| Deleted a namespace by accident | Deleting a namespace removes **everything** inside | Double-check the name before deleting |

---

## 📝 Cheat Sheet

```bash
# Namespaces
kubectl get ns
kubectl create ns dev-ns
kubectl apply -f namespace.yaml
kubectl delete ns dev-ns
kubectl config set-context --current --namespace=dev-ns

# Pods in a namespace
kubectl apply -f pod-limit.yaml
kubectl get pods -n dev-ns
kubectl get pods -A
kubectl describe pod limited-pod -n dev-ns
kubectl logs limited-pod -n dev-ns
kubectl exec -it limited-pod -n dev-ns -- sh

# Resources
kubectl top pod -n dev-ns
kubectl top node
kubectl get pod limited-pod -n dev-ns -o jsonpath='{.status.qosClass}'

# Quota and LimitRange
kubectl apply -f resource-quota.yaml
kubectl apply -f limit-range.yaml
kubectl get resourcequota -n dev-ns
kubectl describe resourcequota -n dev-ns
kubectl get limitrange -n dev-ns
kubectl describe limitrange -n dev-ns
```

| Concept | One-line summary |
|---|---|
| Namespace | Virtual cluster for grouping and isolating resources |
| Request | Minimum reserved, used by scheduler |
| Limit | Maximum allowed, enforced at runtime |
| ResourceQuota | Total cap for a namespace |
| LimitRange | Default, min and max per container |
| CPU over limit | Throttled |
| Memory over limit | OOMKilled |

---

## 🧹 Cleanup

Deleting the namespace removes the Pod, quota and limit range inside it:

```bash
kubectl delete namespace dev-ns
```

Or delete files individually:

```bash
kubectl delete -f pod-limit.yaml
kubectl delete -f resource-quota.yaml
kubectl delete -f limit-range.yaml
kubectl delete -f namespace.yaml
```

---

## 🎯 Practice Challenges

1. Create a second namespace `test-ns` and run a Pod named `limited-pod` in it. Notice the same name works in both namespaces.
2. Change the Pod so that requests **equal** limits and confirm the QoS class becomes `Guaranteed`.
3. Remove the `resources` section and confirm the QoS class is `BestEffort`.
4. Apply `resource-quota.yaml`, then try creating 6 Pods and observe the error on the 6th.
5. Create a Pod requesting `memory: 2Gi` and observe the `exceeded quota` error.
6. Apply `limit-range.yaml`, create a Pod **without** resources, and check that defaults were injected with `kubectl describe pod`.
7. Set `kubectl config set-context --current --namespace=dev-ns` and run `kubectl get pods` without `-n`.

---

## 📖 Further Reading

- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Pod QoS Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

---

⭐ If this helped you, give the repo a star and happy learning! 🚀