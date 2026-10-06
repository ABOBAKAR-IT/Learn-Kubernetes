# Nodes

> Status: 📘 Notes ready

## Idea

A **Node** is a machine (VM or physical) in the cluster. Pods run on nodes.

| Node type | Runs | Notes |
|---|---|---|
| **Control plane node** | API server, etcd, scheduler, controller manager | Usually tainted so your app Pods do not land here |
| **Worker node** | Your Pods | Where the real work happens |

Every worker node runs three things:

| Component | Job |
|---|---|
| **kubelet** | Agent that takes Pod specs from the API server and keeps those Pods running |
| **kube-proxy** | Programs network rules so Services reach Pods |
| **Container runtime** (containerd) | Actually runs the containers |

## Inspect nodes

```bash
kubectl get nodes
kubectl get nodes -o wide                    # IPs, OS, kernel, runtime version
kubectl describe node <node-name>
kubectl top node                             # needs metrics-server
kubectl get node <node-name> -o yaml
kubectl get pods -A -o wide --field-selector spec.nodeName=<node-name>   # Pods on one node
```

### What to read in `describe node`

| Section | Meaning |
|---|---|
| **Conditions** | `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable` |
| **Capacity** | Total CPU, memory, Pod slots on the machine |
| **Allocatable** | What the scheduler can actually hand out to Pods (capacity minus system reserve) |
| **Allocated resources** | Sum of requests and limits of Pods already on the node |
| **Non-terminated Pods** | Every Pod on this node |
| **Events** | Recent node problems |

## Node status and heartbeats

- `Ready`: healthy, accepts Pods.
- `NotReady`: the control plane has stopped hearing from the kubelet. After a timeout its Pods are evicted and rescheduled.
- Each kubelet renews a small **Lease** object, which is how the control plane knows a node is alive.

```bash
kubectl get leases -n kube-node-lease
```

## Node labels

Nodes carry labels you can use for scheduling:

```bash
kubectl get nodes --show-labels
# common ones: kubernetes.io/hostname, kubernetes.io/os, kubernetes.io/arch,
#              node-role.kubernetes.io/control-plane
kubectl label node <node-name> disktype=ssd
kubectl get nodes -l disktype=ssd
```

## Maintenance: cordon, drain, uncordon

```bash
kubectl cordon <node>        # mark unschedulable; existing Pods keep running
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # evict Pods safely
# ...do maintenance (patch, reboot)...
kubectl uncordon <node>      # accept Pods again
```

```
cordon   = "no new Pods here"
drain    = "no new Pods AND move the existing ones elsewhere"
uncordon = "open for business again"
```

`drain` respects PodDisruptionBudgets and skips DaemonSet Pods (they would be recreated immediately).

## Lab (needs a multi-node cluster, see [cluster.md](cluster.md))

```bash
kubectl get nodes -o wide
kubectl create deployment web --image=nginx:1.27 --replicas=4
kubectl get pods -o wide                            # Pods spread across workers

kubectl cordon <one-worker>
kubectl scale deployment web --replicas=6
kubectl get pods -o wide                            # new Pods avoid the cordoned node

kubectl drain <one-worker> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide                            # its Pods moved to other nodes

kubectl uncordon <one-worker>
kubectl delete deployment web
```

## Gotchas

- Single-node minikube has just one node doing both jobs. `drain` there will evict everything.
- Pods are **not** moved back automatically after `uncordon`.
- `Allocatable` is smaller than `Capacity`. Plan with allocatable.

## Check yourself

1. Which three components run on every worker node?
2. What is the difference between `cordon` and `drain`?
3. What do `Capacity` and `Allocatable` mean?
4. How does the control plane know a node is alive?

<details>
<summary>Answers</summary>

1. kubelet, kube-proxy, container runtime
2. `cordon` only stops new Pods. `drain` also evicts the existing Pods so they reschedule elsewhere
3. Capacity is the machine's total. Allocatable is what is left for Pods after system reserves
4. The kubelet renews a Lease object (and reports node status) regularly

</details>