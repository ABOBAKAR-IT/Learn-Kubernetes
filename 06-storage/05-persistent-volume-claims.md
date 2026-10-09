# Persistent Volume Claims (PVC)

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/06-storage/manifests/`](../labs/06-storage/manifests/) · Full walkthrough: [`labs/06-storage`](../labs/06-storage/)

[← Persistent Volumes](persistent-volumes.md) | [README](README.md) | [Storage Classes →](storage-classes.md)

---

## Idea

A **PersistentVolumeClaim (PVC)** is a developer's **request for storage**: how much, how it is accessed, and what class. Kubernetes finds (or creates) a matching PV and **binds** them one to one. The Pod then uses the **claim**, never the PV.

```
Pod ──volumes: persistentVolumeClaim: claimName──► PVC ══binds══► PV ──► real storage
```

- A PVC is **namespaced**. The Pod must be in the **same namespace**.
- Developers ask for storage without knowing where it comes from (EBS, NFS, local disk...).

## YAML explained

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

| Field | Meaning |
|---|---|
| `accessModes` | Required mode(s). The PV must support them |
| `resources.requests.storage` | Minimum size. The bound PV is **at least** this large |
| `storageClassName` | Which class (see the three states below) |
| `volumeMode` | `Filesystem` (default) or `Block` |
| `volumeName` | Bind to **one named PV** |
| `selector` | Only PVs with matching labels |
| `dataSource` | Create the volume from a snapshot or another PVC (clone) |

### The three states of `storageClassName`

| What you write | Meaning |
|---|---|
| **Field missing** | Use the cluster's **default** StorageClass (if one exists) → usually **dynamic provisioning** |
| **`storageClassName: ""`** | **No class**: bind only to a PV that has no class (static only) |
| **`storageClassName: fast`** | Use exactly that class |

This is the source of many "my PVC ignored my PV" problems: a missing field is not the same as an empty string.

## Using a claim in a Pod

```yaml
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: storage-volume
          mountPath: /data
          # readOnly: true
  volumes:
    - name: storage-volume
      persistentVolumeClaim:
        claimName: demo-pvc
        # readOnly: true
```

In a Deployment, **all replicas share the same claim**. With RWO storage, replicas on different nodes cannot all mount it. For one volume **per replica**, use a StatefulSet's `volumeClaimTemplates` (see [StatefulSets](../03-kubernetes-workloads/statefulsets.md)).

## How binding works

A claim binds to a PV only if **all** of these match:

| Rule | Detail |
|---|---|
| Class | Same `storageClassName` (a missing field means the default class) |
| Access modes | The PV supports the requested modes |
| Size | PV capacity ≥ request |
| Optional | `selector` labels, `volumeName` |

If a StorageClass can create volumes, a **new PV is provisioned** to fit. Otherwise the claim waits.

## PVC phases

| Phase | Meaning |
|---|---|
| `Pending` | No match yet (no PV, provisioning in progress, or waiting for a consumer) |
| `Bound` | Bound to a PV |
| `Lost` | The PV is gone and the claim lost its backing storage |

```bash
kubectl get pvc
kubectl describe pvc demo-pvc          # the Events explain WHY it is Pending
```

## PVC protection

A PVC that a **Pod is using** cannot be removed immediately. Deleting it leaves it `Terminating` until the last Pod using it is gone. This prevents data loss under a running app.

## Growing a volume

If the StorageClass has `allowVolumeExpansion: true`, edit the **request** upward (shrinking is not allowed):

```bash
kubectl patch pvc demo-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl get pvc demo-pvc -w
kubectl describe pvc demo-pvc          # condition FileSystemResizePending, then done
```

Many CSI drivers (such as the EBS CSI driver) expand online. minikube's default hostPath provisioner does not support resizing, so try this on a cloud cluster.

## Clones and snapshot restores

```yaml
spec:
  dataSource:
    kind: PersistentVolumeClaim        # clone another PVC (same namespace, CSI drivers only)
    name: demo-pvc
  # or:  kind: VolumeSnapshot, apiGroup: snapshot.storage.k8s.io
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

## Lab

```bash
cd labs/06-storage/manifests
kubectl apply -f pv.yaml                               # demo-pv: 1Gi, class "manual"
```

**1. The three `storageClassName` states side by side**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-default-class        # field missing
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-no-class             # empty string
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ""
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-manual               # named class that matches demo-pv
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: manual
  resources:
    requests:
      storage: 100Mi
EOF
kubectl get pvc -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLASS:.spec.storageClassName,VOLUME:.spec.volumeName
```

Expected on minikube:

- `claim-default-class`: **default class was filled in** (`standard`), a new PV was created and bound.
- `claim-no-class`: `Pending`, because no static PV without a class exists.
- `claim-manual`: bound to **`demo-pv`** (a 1Gi PV can serve a 100Mi request).

```bash
kubectl describe pvc claim-no-class | tail -5          # the Event says why it waits
kubectl delete pvc claim-default-class claim-no-class claim-manual
```

**2. A claim that can never bind**

```bash
kubectl apply -f pvc-pending.yaml                      # asks for 5Gi in class "manual", the PV has 1Gi
kubectl get pvc big-claim                              # Pending
kubectl describe pvc big-claim | tail -4
kubectl delete -f pvc-pending.yaml
```

**3. PVC protection**

```bash
kubectl apply -f pvc.yaml -f pod-with-volume.yaml
kubectl delete pvc demo-pvc --wait=false
kubectl get pvc demo-pvc                               # Terminating: a Pod is still using it
kubectl delete pod pod-with-storage
kubectl get pvc                                        # gone now
```

**4. Static binding with `volumeName`** (optional)

```bash
kubectl patch pv demo-pv -p '{"spec":{"claimRef":null}}'
kubectl apply -f pvc.yaml
kubectl get pvc demo-pvc -o jsonpath='{.spec.volumeName}{"\n"}'     # demo-pv
```

**Cleanup**

```bash
kubectl delete -f pvc.yaml -f pv.yaml --ignore-not-found
minikube ssh "sudo rm -rf /mnt/data"
```

## Gotchas

- **Missing `storageClassName` ≠ `""`.** Missing means the default class, empty means no class.
- A claim stays **`Pending` with `WaitForFirstConsumer`** until a Pod uses it. That is normal, not an error.
- A PVC and its Pod must be in the **same namespace**.
- Deleting a claim for a `Delete`-policy PV **destroys the data**.
- A claim cannot **shrink**.
- All replicas of a Deployment share one claim, which is a problem for RWO volumes across nodes.

## Check yourself

1. What does a PVC represent, and who creates it?
2. What is the difference between omitting `storageClassName` and setting it to `""`?
3. Name three things that must match for a PVC to bind to a PV.
4. A PVC is `Pending`. Where do you look?
5. What happens when you delete a PVC that a running Pod uses?
6. How do you give each replica its own volume?

<details>
<summary>Answers</summary>

1. A request for storage (size, access mode, class). The developer
2. Omitting uses the default StorageClass (often dynamic provisioning). `""` means no class (static PVs only)
3. Class, access modes, and PV size at least the request (or `selector`/`volumeName` if set)
4. `kubectl describe pvc <name>`, the Events
5. It stays `Terminating` until the Pod stops using it (PVC protection)
6. Use a StatefulSet with `volumeClaimTemplates`

</details>

---

[← Persistent Volumes](persistent-volumes.md) | [README](README.md) | [Storage Classes →](storage-classes.md)