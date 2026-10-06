# Kubernetes Volumes: From Zero to PV and PVC

Learn volumes in 5 steps. Each step answers one question and has a lab you can run on **minikube**.

```bash
minikube start --driver=docker
kubectl get nodes
```

All manifests are in [`manifests/`](manifests/). They are also shown inline below.

---

## Topic directory

| Step | Topic | Question it answers |
|---|---|---|
| 0 | [Why volumes exist](#0-why-volumes-exist) | Why does my data disappear? |
| 1 | [emptyDir](#1-emptydir-scratch-space-for-a-pod) | How do containers in a Pod share temporary files? |
| 2 | [hostPath](#2-hostpath-a-folder-on-the-node) | How do I use a folder from the node? (course demo) |
| 3 | [PV, PVC, StorageClass](#3-pv-pvc-and-storageclass-the-persistent-trio) | How do I get storage that outlives Pods? |
| 4 | [Static provisioning lab](#4-lab-static-provisioning-pv--pvc--pod) | Hands-on with your own `pv.yaml`, `pvc.yaml`, Pod |
| 5 | [Dynamic provisioning lab](#5-lab-dynamic-provisioning) | Who creates the PV automatically? |
| 6 | [Access modes, reclaim policy, lifecycle](#6-access-modes-reclaim-policy-and-pv-lifecycle) | What rules control sharing and cleanup? |
| 7 | [Volumes with Deployments, ConfigMaps, EKS](#7-volumes-in-the-real-world) | How is this used in real work? |
| 8 | [Bugs in the original files](#8-bugs-found-in-the-original-files) | Why did my YAML fail? |
| 9 | [Troubleshooting](#9-troubleshooting) | Something is `Pending`. Now what? |
| 10 | [Cheat sheet](#10-cheat-sheet) | Quick review |
| 11 | [Quiz](#11-quiz) | Test yourself |
| 12 | [Cleanup](#12-cleanup) | Remove everything |

---

## 0. Why volumes exist

A container's filesystem is **temporary**. Data is lost in two situations:

| Event | Data in the container filesystem |
|---|---|
| Container crashes and restarts | **Lost** (a fresh filesystem from the image) |
| Pod is deleted or rescheduled | **Lost** |

A **volume** is a directory that lives *outside* the container's own filesystem and is mounted into it. Different volume types survive different events:

```
                         survives container   survives Pod     survives node
                         restart?             deletion?        loss?
container filesystem         no                   no               no
emptyDir                     YES                  no               no
hostPath                     YES                  YES (same node)  no
PersistentVolume (PVC)       YES                  YES              YES (with network storage)
```

The one sentence to remember:

> **The longer you need the data to live, the further away from the container it must be stored.**

### The two parts of every volume

A volume always has two parts in the Pod spec:

```yaml
spec:
  volumes:                      # 1) DECLARE: what volume exists (Pod level)
  - name: my-vol
    emptyDir: {}
  containers:
  - name: app
    volumeMounts:               # 2) MOUNT: where it appears inside the container
    - name: my-vol              #    must match the volume name above
      mountPath: /data
```

If you only do one of the two, it fails. The **`name`** is the link between them.

---

## 1. emptyDir: scratch space for a Pod

**Idea:** an empty directory created when the Pod starts on a node, deleted when the Pod is removed. All containers in the Pod can mount it.

- Survives **container restarts**.
- Does **not** survive **Pod deletion**.
- Uses: scratch space, caches, sharing files between sidecar containers.

#### File: `emptydir-demo.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "date >> /cache/log.txt; cat /cache/log.txt; sleep 15"]
      volumeMounts:
        - name: cache-volume
          mountPath: /cache
  volumes:
    - name: cache-volume
      emptyDir: {}
```

The container adds a line to `/cache/log.txt`, prints the file, sleeps 15 seconds, then exits. Kubernetes restarts it (default `restartPolicy: Always`), so we can watch the file survive restarts.

### Lab 1

```bash
kubectl apply -f manifests/emptydir-demo.yaml
kubectl logs emptydir-demo                 # 1 line

# wait about 40 seconds, the container restarts on its own
kubectl get pod emptydir-demo              # RESTARTS is now 1 or 2
kubectl logs emptydir-demo                 # more lines: the file survived the restart

# now delete the whole Pod and recreate it
kubectl delete pod emptydir-demo
kubectl apply -f manifests/emptydir-demo.yaml
kubectl logs emptydir-demo                 # back to 1 line: new Pod = new emptyDir
```

**What you learned:** `emptyDir` lives as long as the **Pod**, not the container.

Optional memory-backed version (fast, counts against memory limits):

```yaml
emptyDir:
  medium: Memory
  sizeLimit: 64Mi
```

---

## 2. hostPath: a folder on the node

**Idea:** mount a directory that already exists on the **node** (the machine running the Pod) into the container.

- Survives Pod deletion (the folder stays on the node).
- Tied to **one node**. If the Pod moves to another node, it sees a different folder.
- Security risk in real clusters (containers can touch the node's files). Use for learning, local clusters and node agents only.

This is the LFS158 demo: **two containers in one Pod share one volume**. The `debian` container writes a web page and the `nginx` container serves it.

#### File: `app-blue-shared-vol.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: blue-app
  name: blue-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blue-app
  template:
    metadata:
      labels:
        app: blue-app
        type: canary
    spec:
      volumes:
        - name: host-volume
          hostPath:
            path: /home/docker/blue-shared-volume
            type: DirectoryOrCreate
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: host-volume
              mountPath: /usr/share/nginx/html
        - name: debian
          image: debian
          volumeMounts:
            - name: host-volume
              mountPath: /host-vol
          command: ["/bin/sh", "-c", "echo Welcome to BLUE App! > /host-vol/index.html ; sleep infinity"]
```

Changes from the course text:

- Removed `creationTimestamp: null`, `strategy: {}` and `status: {}`. They are leftovers from `kubectl ... --dry-run=client -o yaml` and do nothing.
- Added `type: DirectoryOrCreate` so the folder is created if it does not exist.

### How it works

```
              ┌──────────────── Pod blue-app ────────────────┐
              │  debian container          nginx container   │
              │  writes /host-vol/         serves            │
              │  index.html                /usr/share/nginx/html
              │        └────── volume "host-volume" ─────────┘│
              └───────────────────┬──────────────────────────┘
                                  ▼
                  node folder /home/docker/blue-shared-volume
```

### Lab 2

```bash
kubectl apply -f manifests/app-blue-shared-vol.yaml
kubectl get pods -l app=blue-app                       # READY 2/2

kubectl port-forward deploy/blue-app 8080:80           # keep running, open a second terminal
curl http://localhost:8080                             # Welcome to BLUE App!

# the file really lives on the node
minikube ssh "cat /home/docker/blue-shared-volume/index.html"

# delete the Pod: the Deployment makes a new one, same folder, same page
kubectl delete pod -l app=blue-app
kubectl get pods -l app=blue-app
```

**What you learned:**

1. Volumes are declared once at Pod level and mounted by **any** container in the Pod, at any path.
2. `hostPath` data lives on the node, which is why it is not a real answer for production.

---

## 3. PV, PVC and StorageClass: the persistent trio

Real apps (databases, uploads) need storage that is **independent of any Pod and any node**. Kubernetes splits this into three objects so that developers do not need to know storage details.

### The restaurant analogy

| Kubernetes | Restaurant | Who creates it |
|---|---|---|
| **PersistentVolume (PV)** | A table that exists in the restaurant | Admin (or automatically) |
| **PersistentVolumeClaim (PVC)** | "A table for 1, please" (a request) | Developer |
| **StorageClass (SC)** | A restaurant that **builds a table on demand** | Admin |
| **Pod** | The guest who sits at the table | Developer |

### The chain

```
Pod ──volumeMounts──► volume (name) ──persistentVolumeClaim──► PVC ──binds to──► PV ──backed by──► real storage
                                      claimName: demo-pvc      demo-pvc        demo-pv           hostPath / EBS / NFS
```

The Pod never mentions the PV. It only knows the **claim name**.

### Binding rules (how a PVC finds its PV)

A PVC binds to a PV only when **all** of these match:

| Rule | Detail |
|---|---|
| `storageClassName` | Same on both (an empty string matches only PVs without a class) |
| `accessModes` | The PV supports the modes the PVC requests |
| `capacity` | PV size is **greater than or equal to** the request |
| selector / `volumeName` | Only if you set them |

Binding is **one to one**. A bound PV is exclusive to that PVC, even if the PV is bigger than the request.

### Two ways to get a PV

| | Static provisioning | Dynamic provisioning |
|---|---|---|
| Who creates the PV | An admin, by hand | Kubernetes, from a **StorageClass** |
| Flow | PV exists → PVC binds | PVC created → PV is built automatically |
| Where used | Learning, special disks | Almost all real clusters |

---

## 4. Lab: static provisioning (PV → PVC → Pod)

This is the corrected version of your own files. Order matters: **PV, then PVC, then Pod**.

#### File: `pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```

| Field | Meaning |
|---|---|
| `capacity.storage` | Size this PV offers |
| `accessModes` | How it can be mounted (see section 6) |
| `persistentVolumeReclaimPolicy: Retain` | Keep the data after the claim is deleted |
| `storageClassName: manual` | A label used for matching. `manual` is only a name; no StorageClass object is needed |
| `hostPath` | The real storage, a folder on the node |

#### File: `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

The PVC says what it needs (size, mode, class). It never names the PV. Kubernetes finds a match.

#### File: `pod-with-volume.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-storage
spec:
  containers:
    - name: volume-container
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: storage-volume
          mountPath: /data
  volumes:
    - name: storage-volume
      persistentVolumeClaim:
        claimName: demo-pvc
```

### Lab 3 (run step by step)

**Step 1: create the PV and watch its status**

```bash
kubectl apply -f manifests/pv.yaml
kubectl get pv
```

Expected: `demo-pv   1Gi   RWO   Retain   Available   manual`

**Step 2: create the PVC and watch both bind**

```bash
kubectl apply -f manifests/pvc.yaml
kubectl get pv,pvc
```

Expected: PV `STATUS` is `Bound` and its `CLAIM` shows `default/demo-pvc`. PVC `STATUS` is `Bound` with `VOLUME` = `demo-pv`.

**Step 3: run a Pod that uses the claim**

```bash
kubectl apply -f manifests/pod-with-volume.yaml
kubectl get pod pod-with-storage
kubectl describe pod pod-with-storage       # see "Volumes:" → ClaimName: demo-pvc
```

**Step 4: write data**

```bash
kubectl exec -it pod-with-storage -- sh
echo "Abobakar was here" > /data/note.txt
cat /data/note.txt
exit
```

**Step 5: the proof, delete the Pod and bring it back**

```bash
kubectl delete pod pod-with-storage
kubectl apply -f manifests/pod-with-volume.yaml
kubectl exec pod-with-storage -- cat /data/note.txt     # still there!
minikube ssh "cat /mnt/data/note.txt"                   # also visible on the node
```

**What you learned:** the Pod died, but the data lived on because it was stored in the PV, not in the Pod.

---

## 5. Lab: dynamic provisioning

minikube ships with a **default StorageClass**:

```bash
kubectl get storageclass
```

Expected: a class named `standard (default)` with provisioner `k8s.io/minikube-hostpath`.

Now create a PVC **without** any PV. Note: **no `storageClassName`**, so the default class is used.

#### File: `pvc-dynamic.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

#### File: `pod-dynamic.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-dynamic
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: dynamic-pvc
```

### Lab 4

```bash
kubectl apply -f manifests/pvc-dynamic.yaml
kubectl apply -f manifests/pod-dynamic.yaml
kubectl get pv,pvc
```

Expected: a **new PV** you never wrote, named like `pvc-3f1c...`, bound to `dynamic-pvc`.

```bash
kubectl exec pod-dynamic -- sh -c 'echo dynamic > /data/hello.txt && cat /data/hello.txt'
kubectl get pv -o wide
```

Some clusters (for example kind) use `volumeBindingMode: WaitForFirstConsumer`. There the PVC stays `Pending` until a Pod uses it. That is normal.

### Why your original `pvc.yaml` did not bind to `demo-pv`

Your PVC had no `storageClassName`. On a cluster with a default StorageClass, Kubernetes **fills in the default class**, so the PVC asks for class `standard` while your PV has no class. They do not match, so the PVC gets a **brand-new dynamically created PV** and `demo-pv` stays `Available`.

```bash
kubectl get pvc demo-pvc -o yaml | grep -i storageClass
```

The fix is to set the same `storageClassName` on both (we used `manual`). To explicitly say "no class, only static PVs", use `storageClassName: ""` on the PVC.

---

## 6. Access modes, reclaim policy and PV lifecycle

### Access modes

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by **one node** (several Pods on that node can share it) |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read-write by many nodes (needs NFS, EFS, CephFS...) |
| `ReadWriteOncePod` | RWOP | Read-write by exactly **one Pod** |

Whether a mode works depends on the storage backend. A block disk such as AWS EBS only supports RWO.

### Reclaim policy: what happens to the PV when the PVC is deleted

| Policy | Result |
|---|---|
| `Retain` | PV and data are kept. PV becomes `Released` and needs manual cleanup before reuse |
| `Delete` | PV **and the underlying storage** are deleted (the default for dynamic PVs) |
| `Recycle` | Deprecated, do not use |

### PV lifecycle

```
Available ──PVC binds──► Bound ──PVC deleted──► Released ──(Retain: admin cleans up)──► Available
                                                    └──(Delete)──► storage removed
                                                    └──(error)───► Failed
```

### Lab 5: see `Released`

```bash
kubectl delete pod pod-with-storage
kubectl delete pvc demo-pvc
kubectl get pv                                  # demo-pv is Released, data still on the node
minikube ssh "cat /mnt/data/note.txt"           # still there

# make the PV usable again (clear the old claim reference)
kubectl patch pv demo-pv -p '{"spec":{"claimRef":null}}'
kubectl get pv                                  # Available again
```

---

## 7. Volumes in the real world

### A Deployment using a PVC

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-app:1.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/app
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: demo-pvc
```

Gotcha: **all replicas share the same PVC**. With an RWO volume, replicas on different nodes cannot mount it. For one volume per replica (databases), use a **StatefulSet** with `volumeClaimTemplates`.

### ConfigMap and Secret as volumes

Config files can be mounted into a Pod as files:

```yaml
volumes:
  - name: config-vol
    configMap:
      name: app-config
  - name: secret-vol
    secret:
      secretName: app-secret
```

### Volume types at a glance

| Type | Lifetime | Typical use |
|---|---|---|
| `emptyDir` | Pod | Scratch, cache, sidecar sharing |
| `hostPath` | Node | Node agents, local learning |
| `persistentVolumeClaim` | Independent | Databases, uploads |
| `configMap` / `secret` | Object | Config files, credentials |
| `nfs` and others | Remote | Shared network storage |

### On AWS EKS

- Install the **EBS CSI driver**. A StorageClass (for example `gp3`) then creates EBS volumes dynamically.
- EBS volumes are **RWO** and tied to one **Availability Zone**.
- For shared read-write storage across nodes, use **EFS** with the EFS CSI driver (RWX).
- Volume plugins are now provided by **CSI drivers** (Container Storage Interface), not built into Kubernetes.

---

## 8. Bugs found in the original files

| # | File | Problem | Fix |
|---|---|---|---|
| 1 | `pod-with-colume.yaml` | `volume:` is not a valid field | `volumes:` (plural). The error is `unknown field "spec.volume"` |
| 2 | Pod | `command: ["sh","-c","sleep","3600"]` runs `sleep` with no argument, so it exits immediately and the Pod crash-loops | `["sh","-c","sleep 3600"]` (the whole command is **one** string after `-c`) |
| 3 | PVC | No `storageClassName`, so the default StorageClass takes over and `demo-pv` is never used | Same `storageClassName` on PV and PVC |
| 4 | File names | File is named `pod-with-colume.yaml`, README applies `pod-with-volume.yaml` | One consistent name |
| 5 | PV | `hostPath` without `type` | Add `type: DirectoryOrCreate` |
| 6 | Pod | `image: busybox` has no tag (pulls `latest`) | Pin a version, `busybox:1.36` |
| 7 | README | No verification of persistence and no cleanup | Delete and recreate the Pod, then clean up (sections 4 and 12) |

The `sh -c` rule explained:

```
["sh", "-c", "sleep 3600"]      correct: shell runs the string "sleep 3600"
["sh", "-c", "sleep", "3600"]   wrong:   shell runs "sleep" and "3600" becomes $0
["sleep", "3600"]               also correct: no shell needed
```

---

## 9. Troubleshooting

| Symptom | Likely cause | Check or fix |
|---|---|---|
| PVC `Pending` | No PV matches (class, size, or access mode), or `WaitForFirstConsumer` | `kubectl describe pvc <name>` → Events |
| Pod `Pending`, event `unbound immediate PersistentVolumeClaims` | The claim is not bound yet | Fix the PVC first |
| PVC bound to a PV you did not create | Default StorageClass was applied | Set `storageClassName` explicitly |
| `unknown field "spec.volume"` | Typo, must be `volumes` | Fix the YAML |
| Pod `CrashLoopBackOff` with busybox | Command exits immediately | Use `sleep 3600` as one string |
| Pod `ContainerCreating` for long | Volume cannot be mounted | `kubectl describe pod` → Events |
| `Permission denied` writing to the volume | Container runs as non-root | Set `securityContext.fsGroup` or fix folder permissions |
| PV `Released`, new PVC will not bind | Old `claimRef` remains | `kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'` |
| Data missing after `minikube delete` | hostPath data lived inside the minikube node | Expected, the node was deleted |

Debug flow:

```bash
kubectl get pv,pvc
kubectl describe pvc <name>
kubectl describe pod <name>        # Events at the bottom
kubectl get storageclass
```

---

## 10. Cheat sheet

```bash
# Inspect
kubectl get pv
kubectl get pvc
kubectl get pv,pvc
kubectl get storageclass            # or: kubectl get sc
kubectl describe pv <name>
kubectl describe pvc <name>

# Create / delete
kubectl apply -f manifests/pv.yaml
kubectl delete pvc <name>
kubectl delete pv <name>

# Look inside
kubectl exec -it <pod> -- sh
kubectl exec <pod> -- ls /data
minikube ssh "ls /mnt/data"

# Reuse a Released PV
kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'
```

Short names: `pv`, `pvc`, `sc`.

Volume YAML skeleton:

```yaml
spec:
  volumes:
    - name: <volume-name>
      persistentVolumeClaim:
        claimName: <pvc-name>
  containers:
    - name: <c>
      volumeMounts:
        - name: <volume-name>     # same name
          mountPath: /path/inside
```

---

## 11. Quiz

1. What happens to data in a container's own filesystem when the container restarts?
2. What is the difference between `emptyDir` and `hostPath` lifetimes?
3. In the Pod spec, which field links `volumeMounts` to `volumes`?
4. Who creates a PV, a PVC, and a StorageClass?
5. Does a Pod reference a PV directly?
6. Name three rules a PVC and a PV must satisfy to bind.
7. Why did the original `pvc.yaml` not bind to `demo-pv` on minikube?
8. What is the difference between static and dynamic provisioning?
9. What happens to a PV with `Retain` when its PVC is deleted?
10. Why is `ReadWriteOnce` a problem for a Deployment with 3 replicas on different nodes?
11. Why is `["sh","-c","sleep","3600"]` wrong?
12. Why is `hostPath` not recommended in production?

<details>
<summary>Answers</summary>

1. It is lost; the container starts from the image's fresh filesystem
2. `emptyDir` lives as long as the Pod. `hostPath` lives on the node's disk beyond the Pod, but only on that node
3. The volume `name`
4. PV and StorageClass: an admin (or the cloud/CSI driver). PVC: the developer
5. No. It references the **PVC** by `claimName`
6. Same `storageClassName`, supported access mode, PV capacity at least the requested size
7. It had no `storageClassName`, so the default StorageClass was applied, which does not match the PV's class
8. Static: an admin creates the PV by hand. Dynamic: a StorageClass creates the PV when a PVC asks
9. The PV and data are kept, and the PV becomes `Released` (manual cleanup needed before reuse)
10. An RWO volume can be mounted read-write by only one node at a time
11. `sh -c` takes one command string. Here `sleep` runs without an argument and exits
12. It is tied to one node and exposes the node's filesystem to containers

</details>

---

## 12. Cleanup

```bash
kubectl delete -f manifests/app-blue-shared-vol.yaml
kubectl delete -f manifests/emptydir-demo.yaml --ignore-not-found
kubectl delete -f manifests/pod-with-volume.yaml --ignore-not-found
kubectl delete -f manifests/pod-dynamic.yaml --ignore-not-found
kubectl delete -f manifests/pvc.yaml --ignore-not-found
kubectl delete -f manifests/pvc-dynamic.yaml --ignore-not-found
kubectl delete -f manifests/pv.yaml --ignore-not-found
kubectl get pv,pvc,pods
```

Data left on the node by `Retain` or `hostPath`:

```bash
minikube ssh "sudo rm -rf /mnt/data /home/docker/blue-shared-volume"
```

---

