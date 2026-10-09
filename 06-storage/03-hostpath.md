# emptyDir

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/06-storage/manifests/`](../labs/06-storage/manifests/)

[← Volumes](volumes.md) | [README](README.md) | [hostPath →](hostpath.md)

---

## Idea

An **emptyDir** is an **empty directory created when the Pod is assigned to a node** and **deleted when the Pod leaves the node**. Every container in the Pod can mount it.

| Event | emptyDir data |
|---|---|
| A container crashes and restarts | **Kept** |
| The Pod is deleted or recreated | **Lost** |
| The Pod is rescheduled to another node | **Lost** (a new empty one is created) |

```
Pod
├── container A ──mount /shared──┐
├── container B ──mount /data────┼── the same emptyDir (on the node's disk, or in memory)
└── init container ──────────────┘
```

## Backing: disk or memory

| Setting | Stored in | Notes |
|---|---|---|
| default | The node's disk | Counts toward the Pod's **ephemeral storage** |
| `medium: Memory` | **tmpfs** (RAM) | Fast, never touches disk, **counts toward the container's memory limit** |

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 500Mi            # the Pod is evicted if it goes over

  - name: ram
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```

## Typical uses

| Use | How |
|---|---|
| Scratch space | Sorting, temporary files, downloads |
| Cache | Rebuilt if lost |
| **Sidecar sharing** | An app writes logs, a sidecar ships them |
| **Init handoff** | An init container prepares files for the app |
| Checkpoints | Long jobs that resume after a **container** restart |

**Not for:** data you cannot afford to lose. Use a PersistentVolume.

## YAML explained

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
        - name: cache-volume        # matches the volume name below
          mountPath: /cache
  volumes:
    - name: cache-volume
      emptyDir: {}                  # {} = defaults (node disk, no size limit)
```

The container appends a line, prints the file, sleeps 15 seconds and exits. Kubernetes restarts it (`restartPolicy: Always`), so the file grows.

## Lab

```bash
cd labs/06-storage/manifests
```

**1. Survives container restarts, not Pod deletion**

```bash
kubectl apply -f emptydir-demo.yaml
kubectl logs emptydir-demo                          # 1 line
# wait about 40 seconds: the container restarts on its own
kubectl get pod emptydir-demo                       # RESTARTS 1 or 2
kubectl logs emptydir-demo                          # more lines: the file survived

kubectl delete pod emptydir-demo
kubectl apply -f emptydir-demo.yaml
kubectl logs emptydir-demo                          # back to 1 line: new Pod, new emptyDir
```

**2. Two containers sharing one emptyDir**

```bash
kubectl apply -f emptydir-shared.yaml
kubectl logs emptydir-shared -c reader --tail=3     # lines written by the OTHER container
kubectl exec emptydir-shared -c writer -- ls -l /shared
kubectl exec emptydir-shared -c reader -- ls -l /data    # the same file, mounted at another path
```

**3. Memory-backed emptyDir**

```bash
kubectl apply -f emptydir-memory.yaml
kubectl logs emptydir-memory                        # filesystem type tmpfs, size about 64M
```

**4. Exceed the size limit (the Pod gets evicted)**

```bash
kubectl apply -f emptydir-sizelimit.yaml            # tries to write 100Mi into a 50Mi emptyDir
kubectl get pod emptydir-sizelimit -w               # within about a minute: Error / Evicted
kubectl describe pod emptydir-sizelimit | grep -i -A2 evicted
```

**Cleanup**

```bash
kubectl delete -f emptydir-demo.yaml -f emptydir-shared.yaml -f emptydir-memory.yaml -f emptydir-sizelimit.yaml --ignore-not-found
```

## Gotchas

- **Lifetime is the Pod, not the container.** A container restart keeps the data, a Pod replacement loses it.
- A **memory-backed** emptyDir uses RAM that counts against the container's memory limit. Filling it can get the container `OOMKilled`.
- Without a `sizeLimit`, a runaway process can fill the **node's disk** and cause evictions.
- The eviction for `sizeLimit` is enforced periodically by the kubelet, so it is not instant.
- A Deployment's replicas each get their **own** emptyDir. They do not share data with each other.

## Check yourself

1. When is an emptyDir created, and when is it deleted?
2. Does an emptyDir survive a container restart? A Pod deletion?
3. What does `medium: Memory` change, and what is the risk?
4. How do two containers in a Pod share files?
5. What happens when a Pod exceeds an emptyDir `sizeLimit`?

<details>
<summary>Answers</summary>

1. Created when the Pod is assigned to a node, deleted when the Pod is removed from the node
2. Yes for a container restart. No for Pod deletion
3. The data lives in tmpfs (RAM), which is fast but counts toward the memory limit, so filling it can cause an OOM kill
4. Both mount the same emptyDir volume (each at its own `mountPath`)
5. The kubelet evicts the Pod

</details>

---

[← Volumes](volumes.md) | [README](README.md) | [hostPath →](hostpath.md)