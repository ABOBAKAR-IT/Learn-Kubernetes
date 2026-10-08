# Pods

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/)

[← README](README.md) | [README](README.md) | [ReplicaSets →](replicasets.md)

---

## Idea

A **Pod** is the smallest workload object in Kubernetes. It is the unit of deployment and represents **one instance of your application**.

A Pod is a logical wrapper around **one or more containers**. It guarantees that its containers:

| Guarantee | Meaning |
|---|---|
| Scheduled together | Always on the same node |
| Share one network namespace | One IP address; containers talk via `localhost` |
| Can mount the same volumes | Shared storage and dependencies |

Think of a Pod as **one small apartment**: the containers are roommates who share the address, the kitchen (volumes) and the building (node).

**Pods are ephemeral and cannot self-heal.** If a Pod dies, nothing recreates it. That is why Pods are run by **controllers** (also called operators): Deployments, ReplicaSets, DaemonSets, Jobs. When a controller manages Pods, the Pod's spec is nested inside it as the **Pod Template**.

## Picture

```
Single-container Pod          Multi-container Pod
┌──────────────────┐          ┌───────────────────────────┐
│ Pod  IP 10.1.0.5 │          │ Pod  IP 10.1.0.6          │
│  ┌────────────┐  │          │ ┌────────┐   ┌──────────┐ │
│  │ nginx :80  │  │          │ │  app   │◄─►│ sidecar  │ │  localhost
│  └────────────┘  │          │ └───┬────┘   └────┬─────┘ │
└──────────────────┘          │     └── shared volume ──┘ │
                              └───────────────────────────┘
```

## YAML explained

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    run: nginx-pod
spec:
  containers:
  - name: nginx-pod
    image: nginx:1.22.1
    ports:
    - containerPort: 80
```

| Field | Required | Meaning |
|---|---|---|
| `apiVersion: v1` | Yes | Pods use the core API, `v1` |
| `kind: Pod` | Yes | The object type |
| `metadata` | Yes | Name, optional labels and annotations |
| `spec` | Yes | The desired state, called the **PodSpec** |
| `containers[].image` | Yes | Image pulled from a registry (here Docker Hub) |
| `containerPort: 80` | No | Port the app uses, exposed to Services and clients later |

What happens after you create it:

1. The scheduler reads `spec` and picks a node.
2. The **kubelet** on that node takes responsibility.
3. The kubelet uses the node's **container runtime** to run the image.
4. The Pod's name and labels are used for workload accounting.

**YAML rules:** indent with **2 spaces**, never TAB. A wrong indent is the number one source of errors.

## Commands

```bash
# Declarative (from a file)
kubectl create -f def-pod.yaml          # create, fails if it already exists
kubectl apply  -f def-pod.yaml          # create or update (preferred)

# Imperative (no file)
kubectl run nginx-pod --image=nginx:1.22.1 --port=80

# Generate a starter YAML without running anything (life-saver)
kubectl run nginx-pod --image=nginx:1.22.1 --port=80 \
  --dry-run=client -o yaml > nginx-pod.yaml

# JSON works too
kubectl run nginx-pod --image=nginx:1.22.1 --port=80 \
  --dry-run=client -o json > nginx-pod.json

# Inspect
kubectl get pods
kubectl get pod nginx-pod -o yaml
kubectl get pod nginx-pod -o json
kubectl describe pod nginx-pod          # read Events at the bottom
kubectl logs nginx-pod

# Delete
kubectl delete pod nginx-pod
```

## Lab 1

> The manifests for this note are in [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/). Run the commands from that folder: `cd labs/02-deployment/manifests`.

```bash
kubectl run nginx-pod --image=nginx:1.22.1 --port=80 --dry-run=client -o yaml > nginx-pod.yaml
cat nginx-pod.yaml                      # read the generated YAML, match it to the table above
kubectl apply -f nginx-pod.yaml
kubectl get pods -o wide                # which node? which IP?
kubectl describe pod nginx-pod          # find: Node, IP, Events (Scheduled, Pulled, Started)
kubectl port-forward pod/nginx-pod 8080:80   # open http://localhost:8080
kubectl delete pod nginx-pod
kubectl get pods                        # it is gone and NOT recreated
```

## Gotchas

- A bare Pod that is deleted or whose node dies stays gone.
- `containerPort` is informational. It does not open or block ports.
- `create` fails if the object exists. `apply` is idempotent.

## Check yourself

1. Which three things do containers in one Pod share?
2. Why are Pods run through controllers?
3. What does `--dry-run=client -o yaml` do?

---

## Deep dive: Pod lifecycle, restarts and graceful shutdown

Pods are born, run, fail and die in a predictable way. Knowing the stages explains most `kubectl get pods` output.

### Pod phases

`kubectl get pod <name> -o jsonpath='{.status.phase}'`

| Phase | Meaning |
|---|---|
| `Pending` | Accepted, but not running yet (waiting to be scheduled, pulling images, init containers running) |
| `Running` | Bound to a node and at least one container is running or starting |
| `Succeeded` | All containers exited with code 0 and will not restart (Jobs) |
| `Failed` | All containers ended and at least one failed |
| `Unknown` | The node cannot be reached |

`CrashLoopBackOff`, `ImagePullBackOff` and `Completed` are **not phases**. They are container state reasons shown in the `STATUS` column.

### Container states

| State | Meaning | Common reasons |
|---|---|---|
| `Waiting` | Not running yet | `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff` |
| `Running` | Executing | n/a |
| `Terminated` | Finished | `Completed`, `Error`, `OOMKilled` |

Read exit codes:

```bash
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].state}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

| Exit code | Meaning |
|---|---|
| `0` | Success |
| `1` | Application error |
| `137` | Killed by SIGKILL (128 + 9): out of memory (`OOMKilled`) or grace period ran out |
| `143` | Stopped by SIGTERM (128 + 15): normal shutdown |

### restartPolicy

Set at Pod level, applies to all containers.

| Value | Restarts when | Used by |
|---|---|---|
| `Always` (default) | Any exit, even success | Deployments, ReplicaSets, DaemonSets, StatefulSets |
| `OnFailure` | Non-zero exit | Jobs |
| `Never` | Never | Jobs, one-off Pods |

Restarts use an increasing back-off (10s, 20s, 40s... up to 5 minutes), which you see as `CrashLoopBackOff`. The timer resets after the container runs cleanly for 10 minutes.

### Init containers

Run **one at a time, in order, before** the app containers. Each must exit 0 or the Pod does not start.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: wait-for-service
      image: busybox:1.36
      command: ["sh", "-c", "until nslookup myservice; do echo waiting for myservice; sleep 2; done"]
  containers:
    - name: app
      image: nginx:1.27
```

Uses: wait for a dependency, run a database migration, download config, fix file permissions. The Pod shows `Init:0/1` while it waits.

### Multi-container patterns

| Pattern | Idea | Example |
|---|---|---|
| **Sidecar** | Helper beside the main app | Log shipper, proxy, config reloader |
| **Ambassador** | Proxy to the outside world | Local proxy to a database cluster |
| **Adapter** | Reshape the app's output | Convert metrics to Prometheus format |

All containers share the network namespace (`localhost`) and can share volumes. Newer clusters support **native sidecars**: an init container with `restartPolicy: Always` that starts first, keeps running, and stops last.

### Termination: what happens on `kubectl delete pod`

```
delete requested
   │
   ├─► Pod marked Terminating, removed from Service endpoints   (in parallel)
   ├─► preStop hook runs (if defined)
   ├─► SIGTERM sent to PID 1 of each container
   ├─► wait up to terminationGracePeriodSeconds (default 30)
   └─► SIGKILL if still running
```

Key points:

1. Your app must **handle SIGTERM** and finish in-flight work.
2. The Pod is removed from Service endpoints at the same time as shutdown starts, so a few requests may still arrive. A short `preStop` sleep gives load balancers time to stop sending traffic.
3. Containers must run as **PID 1 in exec form**, or signals never reach the app.

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      image: my-app:1.0
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]
```

**Node.js and NestJS:**

```dockerfile
# exec form: node is PID 1 and receives SIGTERM
CMD ["node", "dist/main.js"]
# not: CMD npm start   (npm does not forward signals)
```

```ts
// main.ts
const app = await NestFactory.create(AppModule);
app.enableShutdownHooks();   // SIGTERM triggers onModuleDestroy / onApplicationShutdown
await app.listen(3000);
```

Lifecycle hooks: `postStart` (right after the container starts) and `preStop` (right before SIGTERM).

### Lab

> The manifests for this note are in [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/). Run the commands from that folder: `cd labs/02-deployment/manifests`.

**1. Phases and exit codes**

```bash
kubectl run crash --image=busybox:1.36 --restart=Never -- sh -c "echo failing; exit 3"
kubectl get pod crash                                   # Error
kubectl get pod crash -o jsonpath='{.status.phase}{"\n"}'                              # Failed
kubectl get pod crash -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}{"\n"}'   # 3
kubectl delete pod crash
```

**2. Init container**

```bash
kubectl apply -f init-demo.yaml           # the YAML above
kubectl get pod init-demo -w              # Init:0/1, stuck waiting
kubectl create service clusterip myservice --tcp=80:80    # in another terminal
kubectl get pod init-demo                 # PodInitializing, then Running
kubectl delete pod init-demo; kubectl delete svc myservice
```

**3. Graceful vs ignored SIGTERM**

```yaml
# sigterm-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sigterm-ignored
spec:
  terminationGracePeriodSeconds: 10
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: sigterm-handled
spec:
  terminationGracePeriodSeconds: 10
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "trap 'echo got SIGTERM, cleaning up; exit 0' TERM; echo running; while true; do sleep 1; done"]
```

```bash
kubectl apply -f sigterm-demo.yaml
kubectl wait --for=condition=Ready pod --all --timeout=60s
time kubectl delete pod sigterm-handled      # about 1 to 3 seconds
time kubectl delete pod sigterm-ignored      # about 10 seconds: waited the whole grace period, then SIGKILL
```

### Gotchas

- `Completed` is a normal state for Jobs, not an error.
- `Running` does not mean `Ready`. Readiness is a separate condition (module 08).
- `CrashLoopBackOff` is Kubernetes **waiting before the next restart**, not a crash by itself. Read `kubectl logs <pod> --previous`.
- `npm start` or a shell wrapper as PID 1 hides SIGTERM from your app.

### Check yourself

1. Is `CrashLoopBackOff` a Pod phase?
2. What does exit code 137 usually mean?
3. In what order do init containers run?
4. What is the default `terminationGracePeriodSeconds`?
5. Why can `CMD npm start` cause slow shutdowns?

<details>
<summary>Answers</summary>

1. No. It is a container state reason; the phase is still `Running` or `Pending`
2. SIGKILL: out of memory (`OOMKilled`) or the grace period expired
3. One after another, each must succeed before the next, and all finish before app containers start
4. 30 seconds
5. npm does not forward SIGTERM to your app, so Kubernetes waits the full grace period and then kills it

</details>

---

## Deep dive: Resources, QoS classes and a multi-container lab

### Resource requests, limits and QoS classes
```yaml
resources:
  requests: { cpu: 100m, memory: 128Mi }    # used by the scheduler
  limits:   { cpu: 500m, memory: 256Mi }    # enforced by the kubelet
```

| QoS class | When | Eviction order |
|---|---|---|
| **Guaranteed** | requests = limits for every container | Last to be evicted |
| **Burstable** | some requests or limits set | Middle |
| **BestEffort** | no requests or limits | First to be evicted |

Exceeding the **memory limit** gets the container killed (`OOMKilled`, exit code 137). Exceeding the **CPU limit** only slows it down (throttling). More in [07-scheduling](../07-scheduling/).

### Lab: init container + sidecar + shared volume
```yaml
# patterns-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: patterns-demo
spec:
  terminationGracePeriodSeconds: 20
  initContainers:
    - name: init-content
      image: busybox:1.36
      command: ["sh", "-c", "echo 'written by the init container' > /work/index.html; sleep 5"]
      volumeMounts:
        - name: shared
          mountPath: /work
  containers:
    - name: web
      image: nginx:1.27
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "echo preStop running; sleep 5"]
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /shared/heartbeat.log; sleep 5; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

```bash
kubectl apply -f patterns-demo.yaml
kubectl get pod patterns-demo -w                   # Init:0/1 → PodInitializing → Running 2/2

# the init container prepared the file
kubectl exec patterns-demo -c web -- cat /usr/share/nginx/html/index.html

# the sidecar writes into the same volume that web mounts
kubectl exec patterns-demo -c web -- tail -n 3 /usr/share/nginx/html/heartbeat.log

# containers and conditions
kubectl get pod patterns-demo -o jsonpath='{.spec.initContainers[*].name} | {.spec.containers[*].name}{"\n"}'
kubectl get pod patterns-demo -o jsonpath='{.status.qosClass}{"\n"}'      # BestEffort (no requests)
kubectl logs patterns-demo -c web

# shutdown: time it
time kubectl delete pod patterns-demo
```

Expect the delete to take close to the full 20 second grace period. The sidecar's shell loop does not handle SIGTERM, so Kubernetes has to wait and then force-kill it. Fix it by handling the signal (for example with `trap`) or by running a program that exits on SIGTERM.

Break it on purpose: change the init container command to `exit 1` and watch the Pod stay in `Init:Error` / `Init:CrashLoopBackOff` while `web` never starts.

---

[← README](README.md) | [README](README.md) | [ReplicaSets →](replicasets.md)