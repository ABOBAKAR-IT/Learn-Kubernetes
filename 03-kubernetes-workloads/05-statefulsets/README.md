# StatefulSets

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/)

[← DaemonSets](daemonsets.md) | [README](README.md) | [Jobs →](jobs.md)

---

## Idea

A **StatefulSet** runs Pods that need a **stable identity** and **their own persistent storage**. Use it for databases and clustered systems (PostgreSQL, MySQL, MongoDB, Kafka, ZooKeeper, Elasticsearch).

A Deployment treats Pods as **interchangeable cattle**: random names, shared or no storage, any order. A StatefulSet treats each Pod as a **named pet**.

```
Deployment:   web-7d9f8-x2k1  web-7d9f8-p9qz  web-7d9f8-m4tr    random names, replaceable
StatefulSet:  db-0            db-1            db-2              fixed names, fixed order
                │               │               │
              disk-db-0       disk-db-1       disk-db-2         each Pod keeps its own disk
```

## The four guarantees

| Guarantee | What it means |
|---|---|
| **Stable names** | Pods are `<name>-0`, `<name>-1`, `<name>-2`. A replacement for `db-1` is again `db-1` |
| **Stable network identity** | Each Pod has its own DNS name through a **headless Service** |
| **Stable storage** | `volumeClaimTemplates` create one PVC per Pod. It follows the Pod's name and survives Pod deletion |
| **Ordered operations** | Start `0 → 1 → 2`, stop `2 → 1 → 0`, update from the highest number down |

## Deployment vs StatefulSet

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random | `name-0`, `name-1`... |
| Storage | Shared PVC or none | **One PVC per Pod** |
| Startup order | All at once | One by one (default) |
| Network identity | Via a Service (load balanced) | Per-Pod DNS name |
| Scale down | Any Pod | Highest ordinal first |
| Best for | Stateless apps | Databases, queues, clustered apps |

## The headless Service

A StatefulSet needs a **headless Service** (`clusterIP: None`). It has no virtual IP. DNS returns the Pod addresses directly, so each Pod is reachable by name:

```
db-0.db.default.svc.cluster.local     → IP of db-0
db-1.db.default.svc.cluster.local     → IP of db-1
nslookup db                           → the IPs of all Ready Pods
```

The StatefulSet's `serviceName` must name this Service.

## YAML explained

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  clusterIP: None              # headless
  selector:
    app: web
  ports:
    - port: 80
      name: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web             # the headless Service above
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web               # must match selector
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: www                      # matches the template name below
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:        # one PVC is created per Pod from this template
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Mi
```

| Field | Meaning |
|---|---|
| `serviceName` | The headless Service that gives Pods their DNS names |
| `volumeClaimTemplates` | Blueprint for per-Pod PVCs. PVC name = `<template>-<sts>-<ordinal>`, here `www-web-0` |
| `podManagementPolicy` | `OrderedReady` (default, one at a time) or `Parallel` (all at once) |
| `updateStrategy` | `RollingUpdate` (default, highest ordinal first, with optional `partition` for canaries) or `OnDelete` |

## Lab

Needs a default StorageClass (minikube and kind have one). See [Storage](../06-storage/).

```bash
kubectl apply -f web-statefulset.yaml
kubectl get pods -w                                    # web-0, then web-1, then web-2, in order
kubectl get pvc                                        # www-web-0, www-web-1, www-web-2
```

**1. Give each Pod its own data**

```bash
for i in 0 1 2; do
  kubectl exec web-$i -- sh -c "echo 'Hello from web-$i' > /usr/share/nginx/html/index.html"
done
```

**2. Stable DNS names**

```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh -c '
  for i in 0 1 2; do wget -qO- http://web-$i.web; done
  nslookup web'
```

**3. Delete a Pod: same name, same data**

```bash
kubectl get pod web-1 -o wide                          # note the IP
kubectl delete pod web-1
kubectl get pods -w                                    # web-1 comes back, same name
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html   # data still there
kubectl get pod web-1 -o wide                          # the IP may have changed, the name did not
```

**4. Scale down and up: the disks stay**

```bash
kubectl scale statefulset web --replicas=1             # web-2 stops first, then web-1
kubectl get pods
kubectl get pvc                                        # all three PVCs still exist
kubectl scale statefulset web --replicas=3
kubectl exec web-2 -- cat /usr/share/nginx/html/index.html   # old data is back
```

**5. Rolling update from the top**

```bash
kubectl set image statefulset/web nginx=nginx:1.26
kubectl rollout status statefulset/web                 # web-2, then web-1, then web-0
```

**Cleanup. PVCs are NOT deleted with the StatefulSet.**

```bash
kubectl delete statefulset web
kubectl delete service web
kubectl delete pvc -l app=web                          # remove the data on purpose
```

## Gotchas

- **A StatefulSet gives identity and storage, not replication.** It will not copy data between `db-0` and `db-1` or elect a leader. The database software (or an **operator**) must do that.
- Deleting a StatefulSet or scaling it down **keeps the PVCs** by default, so data is safe but disks cost money until you delete them. Newer versions add `persistentVolumeClaimRetentionPolicy` to control this.
- Mounting an empty PV over `/usr/share/nginx/html` hides nginx's default page, so `index.html` must be written first (as in the lab).
- If `web-0` cannot start (for example its PVC is stuck `Pending`), `web-1` never starts under `OrderedReady`.
- Running a database on Kubernetes works, but many teams use a managed database (such as RDS) for production instead.

## Check yourself

1. Name three guarantees a StatefulSet gives.
2. Why does a StatefulSet need a headless Service?
3. What happens to the PVCs when you delete a StatefulSet?
4. In which order does a rolling update replace Pods?
5. Does a StatefulSet replicate data between its Pods?

<details>
<summary>Answers</summary>

1. Stable names, stable network identity, stable per-Pod storage, ordered start/stop/update (any three)
2. To give every Pod its own DNS name (`pod.service.namespace`) instead of one load-balanced IP
3. They are kept. You delete them yourself
4. From the highest ordinal down to 0
5. No. The application or an operator must handle replication

</details>

---

[← DaemonSets](daemonsets.md) | [README](README.md) | [Jobs →](jobs.md)