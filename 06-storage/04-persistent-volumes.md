# Persistent Volumes (PV)

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/06-storage/manifests/`](../labs/06-storage/manifests/) · Full walkthrough: [`labs/06-storage`](../labs/06-storage/)

[← hostPath](hostpath.md) | [README](README.md) | [Persistent Volume Claims →](persistent-volume-claims.md)

---

## Idea

A **PersistentVolume (PV)** is a **piece of storage in the cluster** with its own lifecycle, independent of any Pod. It is a **cluster-scoped** object (no namespace) that describes real storage: a cloud disk, an NFS export, a local disk.

| Role | Object | Created by |
|---|---|---|
| The storage itself | **PV** | An admin, or **automatically** by a StorageClass |
| The request for storage | [PVC](persistent-volume-claims.md) | The developer |
| The recipe for creating PVs on demand | [StorageClass](storage-classes.md) | The admin |

> Analogy: a PV is a **table that exists in a restaurant**. A PVC is the **request** for a table. A StorageClass is a restaurant that **builds a table on demand**.

## YAML explained

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
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
| `capacity.storage` | The size this PV offers |
| `volumeMode` | `Filesystem` (default, mounted as a directory) or `Block` (a raw device) |
| `accessModes` | How it can be mounted (table below) |
| `persistentVolumeReclaimPolicy` | What happens to the PV and data when its claim is deleted |
| `storageClassName` | A label used to **match** claims (here the name `manual` is just a label) |
| the source (`hostPath`, `nfs`, `csi`, `local`...) | The **real** storage |
| `nodeAffinity` | Which nodes can use it (required for `local` volumes) |
| `mountOptions` | Extra mount flags, for example for NFS |

### Other sources

```yaml
# NFS: shared read-write storage
  nfs:
    server: 10.0.0.20
    path: /exports/data
```

```yaml
# CSI: how modern cloud storage is attached (what a StorageClass creates for you)
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc123def4567890
    fsType: ext4
```

On managed clusters you rarely write PVs by hand. StorageClasses create them. The old built-in cloud plugins such as `awsElasticBlockStore` are deprecated in favor of CSI drivers.

## Access modes

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by **one node** (several Pods on that node can use it) |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read-write by many nodes |
| `ReadWriteOncePod` | RWOP | Read-write by exactly **one Pod** |

What works depends on the **backend**:

| Backend | Typical modes |
|---|---|
| AWS EBS | RWO (single Availability Zone) |
| AWS EFS, NFS, CephFS | RWX |
| hostPath, `local` | RWO |

A volume can only be mounted in **one mode at a time**, even if it lists several.

## Reclaim policy

What happens to the PV when its PVC is deleted:

| Policy | Result | Default for |
|---|---|---|
| **`Retain`** | PV becomes `Released`, **data kept**, an admin cleans up | Manually created PVs |
| **`Delete`** | The PV **and the underlying storage** are deleted | **Dynamically provisioned** PVs |
| `Recycle` | Deprecated, do not use | n/a |

Change it on an existing PV (this is how you protect dynamic data before deleting its claim):

```bash
kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

## Lifecycle

```
Available ──PVC binds──► Bound ──PVC deleted──► Released ─┬─ Retain: admin cleans up, clears claimRef ─► Available
   ▲                                                     ├─ Delete: PV and storage removed
   └──────────────────────────────────────────────────── └─ error ─► Failed
```

| Phase | Meaning |
|---|---|
| `Available` | Free, waiting for a claim |
| `Bound` | Attached to a PVC (`claimRef` shows which) |
| `Released` | The claim is gone, but the PV keeps its old `claimRef` and data, so it is **not reusable yet** |
| `Failed` | Automatic reclamation failed |

A **Released** PV does not bind to a new claim until you clear the old reference:

```bash
kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'
```

**PV protection:** a PV that is bound to a claim cannot be deleted right away. It shows `Terminating` until the claim is gone.

## Raw block volumes

For software that wants a raw device (some databases):

```yaml
# PV and PVC both set:  volumeMode: Block
# In the Pod, use volumeDevices instead of volumeMounts:
containers:
  - name: db
    volumeDevices:
      - name: raw
        devicePath: /dev/xvda
```

## Lab

```bash
cd labs/06-storage/manifests
```

**1. Static PV: watch the phases**

```bash
kubectl apply -f pv.yaml
kubectl get pv                                           # demo-pv  1Gi  RWO  Retain  Available  manual
kubectl apply -f pvc.yaml
kubectl get pv,pvc                                       # PV: Bound, CLAIM default/demo-pvc
kubectl apply -f pod-with-volume.yaml
kubectl exec pod-with-storage -- sh -c 'echo "kept by the PV" > /data/note.txt'
kubectl get pv demo-pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.claimRef.name,RECLAIM:.spec.persistentVolumeReclaimPolicy
```

**2. Retain and Released**

```bash
kubectl delete pod pod-with-storage
kubectl delete pvc demo-pvc
kubectl get pv                                           # Released, the claim is gone
minikube ssh "cat /mnt/data/note.txt"                    # the data is still on the node

kubectl apply -f pvc.yaml                                # a new claim does NOT bind to a Released PV
kubectl get pvc                                          # Pending
kubectl delete pvc demo-pvc
kubectl patch pv demo-pv -p '{"spec":{"claimRef":null}}' # clear the old reference
kubectl get pv                                           # Available again
kubectl apply -f pvc.yaml && kubectl get pv,pvc          # Bound again, data still intact
```

**3. PV protection**

```bash
kubectl delete pv demo-pv --wait=false
kubectl get pv                                           # Terminating: still bound to a claim
kubectl delete pvc demo-pvc
kubectl get pv                                           # gone (the object), the data stays on the node
minikube ssh "ls /mnt/data"
```

**4. Protect a dynamic volume with Retain**

```bash
kubectl apply -f pvc-retain-demo.yaml                    # dynamic PVC (default class) plus a Pod
kubectl get pv                                           # new PV with RECLAIM POLICY Delete
PV=$(kubectl get pvc retain-demo-pvc -o jsonpath='{.spec.volumeName}')
kubectl exec retain-demo-pod -- sh -c 'echo precious > /data/important.txt'
kubectl patch pv $PV -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl delete pod retain-demo-pod
kubectl delete pvc retain-demo-pvc
kubectl get pv $PV                                       # Released (it would have been deleted with "Delete")
# minikube's default provisioner uses a hostPath directory on the node:
kubectl get pv $PV -o jsonpath='{.spec.hostPath.path}{"\n"}'
```

**Cleanup**

```bash
kubectl delete pv $PV
kubectl delete -f pvc.yaml -f pv.yaml --ignore-not-found
minikube ssh "sudo rm -rf /mnt/data"
```

## Gotchas

- A **Released** PV with `Retain` is **not** automatically reusable. Clear `claimRef` after cleaning the data.
- `Delete` is the default for dynamic volumes, so **deleting the claim deletes the data**. Patch to `Retain` for anything precious.
- A PV is **cluster-scoped**, but its PVC is **namespaced**.
- A PV bigger than the claim is still given **whole** to that claim.
- Access mode support depends on the backend. A mode in the YAML does not make EBS shareable.

## Check yourself

1. Is a PV namespaced? Who normally creates one?
2. What do `Retain` and `Delete` do when the claim is deleted?
3. List the four PV phases.
4. Why does a new PVC not bind to a `Released` PV?
5. Which access mode lets one Pod, and only one Pod, write?
6. How do you protect a dynamically provisioned volume from deletion?

<details>
<summary>Answers</summary>

1. No, it is cluster-scoped. An admin, or a StorageClass automatically
2. `Retain` keeps the PV and data (PV goes to `Released`). `Delete` removes the PV and the underlying storage
3. Available, Bound, Released, Failed
4. It still holds the old `claimRef`. Clear it after cleanup
5. `ReadWriteOncePod`
6. Patch the PV's `persistentVolumeReclaimPolicy` to `Retain`

</details>

---

[← hostPath](hostpath.md) | [README](README.md) | [Persistent Volume Claims →](persistent-volume-claims.md)