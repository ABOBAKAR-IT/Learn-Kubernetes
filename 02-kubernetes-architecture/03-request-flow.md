# Request Flow: Putting It All Together

> Status: 📘 Notes ready

Read this after the component notes. It follows one command through every part of the cluster.

## 1. `kubectl apply -f deployment.yaml` (3 replicas of nginx)

```
 YOU
  │ 1. kubectl sends the YAML (HTTPS)
  ▼
 kube-apiserver ── authN → authZ → admission → validation
  │ 2. saves the Deployment
  ▼
 etcd ◄── Deployment stored
  │
  │ 3. Deployment controller (controller manager) notices it via watch
  │    → creates a ReplicaSet (stored in etcd)
  │ 4. ReplicaSet controller notices it
  │    → creates 3 Pod objects with no nodeName (stored in etcd)
  │ 5. kube-scheduler sees 3 unscheduled Pods
  │    → filters, scores, binds each to a node (nodeName written via the API)
  ▼
 kubelet on each chosen node (sees "this Pod is mine")
  │ 6. asks the container runtime (CRI): create sandbox, pull image, start container
  │ 7. runs probes, reports Pod status
  ▼
 kube-apiserver ◄── status Running / Ready, saved to etcd
  │ 8. EndpointSlice controller adds Ready Pod IPs to any matching Service
  ▼
 kube-proxy on every node updates its rules → the Service now reaches the new Pods
```

Notice the pattern: **every arrow goes through the API server, and each component only reacts to what it watches.**

## 2. Who wrote what

| Step | Component | Object it created or changed |
|---|---|---|
| 1-2 | kubectl, API server | Deployment |
| 3 | Deployment controller | ReplicaSet |
| 4 | ReplicaSet controller | Pods (no node yet) |
| 5 | Scheduler | Pod `spec.nodeName` |
| 6-7 | Kubelet and runtime | Containers; Pod `status` |
| 8 | EndpointSlice controller, kube-proxy | Endpoints, node network rules |

## 3. Traffic flow to a Service

```
client Pod ─► Service IP ─► (kube-proxy rules, DNAT) ─► Pod IP ─► container
                                                          ▲
                                         Pod-to-Pod networking is the CNI plugin
```

## 4. Other flows

**Delete a Pod owned by a ReplicaSet**

1. kubectl asks the API server to delete it. The Pod gets a deletion timestamp.
2. The kubelet sends SIGTERM, waits the grace period (30s by default), then SIGKILL.
3. The ReplicaSet controller sees fewer Pods than desired and creates a new one.
4. The scheduler and kubelet start it.

**A node dies**

1. Heartbeats (Lease renewals) stop.
2. The node controller marks the node `NotReady` and taints it.
3. After the 300 second toleration, its Pods are evicted.
4. ReplicaSet controllers create replacements and the scheduler places them elsewhere.

## 5. What breaks when each component is down

| Component down | Effect on running apps | Effect on the cluster |
|---|---|---|
| **API server** | Keep running | `kubectl` fails, nothing can change, controllers stall |
| **etcd** (no quorum) | Keep running | No changes accepted, API reads/writes fail |
| **Scheduler** | Keep running | New Pods stay `Pending` |
| **Controller manager** | Keep running | No self-healing, no rollouts, no scaling |
| **kubelet** (one node) | Containers may keep running | That node goes `NotReady`; its Pods are replaced after the timeout |
| **kube-proxy** (one node) | Existing connections keep working | Service changes stop reaching that node |
| **Container runtime** (one node) | Containers on it may stop | Kubelet cannot start or manage Pods there |
| **CNI plugin** | Existing Pods keep their network | New Pods cannot get an IP |

The key idea: the **control plane** (decisions) can be down while the **data plane** (running containers) carries on. You lose the ability to change and heal, not the apps themselves.

## 6. Lab: read the story in the events

```bash
kubectl create deployment flow --image=nginx:1.27 --replicas=2
kubectl get events --sort-by=.metadata.creationTimestamp \
  -o custom-columns=TIME:.metadata.creationTimestamp,SOURCE:.source.component,REASON:.reason,MESSAGE:.message
```

Match every line to a component:

| Event reason | Component that emitted it |
|---|---|
| `ScalingReplicaSet` | deployment-controller |
| `SuccessfulCreate` | replicaset-controller |
| `Scheduled` | default-scheduler |
| `Pulling`, `Pulled`, `Created`, `Started` | kubelet |

If the `SOURCE` column is empty on your version, use `kubectl describe pod <pod>` and read the Events section.

Now watch the delete flow:

```bash
kubectl get pods -w &
kubectl delete pod -l app=flow --wait=false
sleep 15; kill %1
kubectl get deploy,rs,pods -l app=flow
kubectl delete deployment flow
```

## 7. Mini exercise

On paper, draw the boxes (kubectl, API server, etcd, controller manager, scheduler, kubelet, runtime, kube-proxy) and number the arrows for "create a Deployment" without looking at section 1. Then compare.

## Check yourself

1. Which component creates the ReplicaSet when you apply a Deployment?
2. Which component sets `nodeName` on a Pod?
3. Who actually starts the containers?
4. If the scheduler is down, what happens to a new Deployment?
5. If the API server is down, do running Pods stop?
6. Why do all arrows go through the API server?
7. A node dies. How does the app recover?

<details>
<summary>Answers</summary>

1. The Deployment controller in the controller manager
2. The scheduler (through the API server)
3. The kubelet, via the container runtime
4. Pods are created but stay `Pending` (no node assigned)
5. No. They keep running, but nothing can be changed or healed
6. It is the single source of truth, so every change is authenticated, authorized, validated and observable in one place
7. The node controller taints it, Pods are evicted after about 5 minutes, the ReplicaSet creates replacements, and the scheduler places them on healthy nodes

</details>