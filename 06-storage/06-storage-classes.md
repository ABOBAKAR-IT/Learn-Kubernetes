# Storage Classes

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/06-storage/manifests/`](../labs/06-storage/manifests/) · Full walkthrough: [`labs/06-storage`](../labs/06-storage/)

[← Persistent Volume Claims](persistent-volume-claims.md) | [README](README.md)

---

## Idea

A **StorageClass** is a **recipe for creating storage on demand**. When a PVC names a class (or gets the default one), Kubernetes asks the class's **provisioner** to create a real volume and a matching PV. This is **dynamic provisioning**, and it is how almost every real cluster gets storage.

```
PVC "data" (class: gp3, 20Gi)
   │
   ▼
StorageClass gp3 ── provisioner: ebs.csi.aws.com ── parameters: type=gp3
   │
   ▼
Driver creates a 20Gi EBS volume ─► PV is created automatically ─► PVC binds ─► Pod mounts it
```

| | Static | Dynamic |
|---|---|---|
| Who creates the PV | An admin, by hand | The StorageClass provisioner |
| Flow | PV exists first, then the PVC binds | PVC first, then the PV is created |
| Used for | Learning, special pre-existing disks | **Almost everything** |

## YAML explained

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

| Field | Meaning |
|---|---|
| `provisioner` | The driver that creates volumes (CSI driver name) |
| `parameters` | Driver-specific options (disk type, encryption, IOPS...) |
| `reclaimPolicy` | Copied to each created PV: `Delete` (default) or `Retain` |
| `volumeBindingMode` | `Immediate` or `WaitForFirstConsumer` (below) |
| `allowVolumeExpansion` | Whether claims may be grown later |
| `mountOptions` | Mount flags for the volumes |
| `allowedTopologies` | Restrict where volumes may be created (zones) |
| **default annotation** | Marks the class used by PVCs that omit `storageClassName` |

A StorageClass is **cluster-scoped**, and you cannot change its provisioner or parameters after creation. Create a new class instead.

## Binding mode: `Immediate` vs `WaitForFirstConsumer`

| Mode | When the volume is created | Risk |
|---|---|---|
| `Immediate` | As soon as the PVC is created | The volume may land in a **zone where the Pod cannot run** |
| **`WaitForFirstConsumer`** | **After** a Pod using the claim is scheduled | None: the volume is created where the Pod is |

Why it matters with **zonal disks (AWS EBS)**: an EBS volume lives in **one Availability Zone** and can only attach to nodes in that zone. With `Immediate`, a volume might be created in `eu-west-1a` while the scheduler places the Pod in `eu-west-1b`, so the Pod is stuck `Pending`. `WaitForFirstConsumer` lets the scheduler pick the node first.

A claim in this mode shows `Pending` until a Pod uses it. That is **expected**.

## The default class

```bash
kubectl get storageclass                 # (default) marks the default class
```

```bash
# move the default from one class to another
kubectl patch sc standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch sc gp3      -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Keep **one** default. With **no** default, a PVC that omits the class stays `Pending`.

## CSI: how drivers plug in

The **Container Storage Interface (CSI)** is the standard way storage vendors provide drivers. A CSI driver runs as Pods in your cluster:

| Part | Job |
|---|---|
| **Controller plugin** (Deployment) | Create, delete, attach, snapshot, resize volumes in the storage system |
| **Node plugin** (DaemonSet) | Mount the volume on the node where the Pod runs |
| **Sidecars** (`external-provisioner`, `external-attacher`, `external-resizer`, `external-snapshotter`) | Watch Kubernetes objects and call the driver |

Full dynamic provisioning flow:

```
1. PVC created → external-provisioner calls the driver: CreateVolume
2. PV object created and bound to the PVC
3. Pod scheduled → external-attacher attaches the volume to the node
4. Node plugin mounts it → kubelet starts the container
```

The old in-tree plugins (for example `kubernetes.io/aws-ebs`) are deprecated and migrated to CSI.

## Common provisioners

| Environment | Provisioner |
|---|---|
| minikube default | `k8s.io/minikube-hostpath` |
| kind default | `rancher.io/local-path` |
| AWS EBS | `ebs.csi.aws.com` |
| AWS EFS | `efs.csi.aws.com` |
| Google Cloud disks | `pd.csi.storage.gke.io` |
| Azure disks | `disk.csi.azure.com` |

### AWS EKS

- Install the **EBS CSI driver** as an EKS add-on and give it AWS permissions through IRSA or Pod Identity (see [IRSA](../15-aws-eks/irsa.md) and [EBS CSI](../15-aws-eks/ebs-csi.md)):

```bash
aws eks create-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver \
  --service-account-role-arn <role-arn>
```

- Older EKS clusters ship a default class named `gp2` (a legacy provisioner). Create a `gp3` CSI class like the one above and make it the default.
- EBS is **RWO** and **single-AZ**, so use `WaitForFirstConsumer`.
- For **shared read-write** storage across nodes use **EFS**:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0123456789abcdef0
  directoryPerms: "700"
```

## Volume expansion

```yaml
allowVolumeExpansion: true        # on the class
```

Then raise the PVC's `requests.storage`. Shrinking is not possible. Whether it is online (no restart) depends on the driver. EBS supports it.

## Snapshots and clones

With a CSI driver that supports snapshots, plus the snapshot CRDs and controller installed:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snap-1
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: demo-pvc
```

Restore by creating a PVC with a `dataSource` pointing at the snapshot:

```yaml
spec:
  storageClassName: gp3
  dataSource:
    name: db-snap-1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 20Gi
```

Snapshots are the building block for volume-level backups (tools such as Velero use them).

## Lab

```bash
cd labs/06-storage/manifests
```

**1. Inspect your classes**

```bash
kubectl get storageclass
kubectl get sc -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner,RECLAIM:.reclaimPolicy,BINDING:.volumeBindingMode,EXPAND:.allowVolumeExpansion
kubectl describe sc standard
```

Note the binding mode. minikube's default class binds immediately. kind's default class uses `WaitForFirstConsumer`.

**2. Watch dynamic provisioning happen**

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc dynamic-pvc                          # Bound immediately, or Pending (WaitForFirstConsumer)
kubectl apply -f pod-dynamic.yaml
kubectl describe pvc dynamic-pvc | tail -8           # Events: provisioning, then ProvisioningSucceeded
kubectl get pv                                       # a PV you never wrote: pvc-<uid>
```

**3. Try to expand a claim on a class that does not allow it**

```bash
kubectl patch pvc dynamic-pvc -p '{"spec":{"resources":{"requests":{"storage":"1Gi"}}}}'
# rejected: the StorageClass does not support resizing
```

**4. Create your own class and watch delayed binding**

```bash
kubectl apply -f storageclass-custom.yaml            # Retain + WaitForFirstConsumer
kubectl apply -f pvc-custom.yaml
kubectl get pvc custom-pvc                           # Pending: no consumer yet (if the provisioner supports it)
kubectl apply -f pod-custom.yaml
kubectl get pvc,pv                                   # now Bound
kubectl get pv -o custom-columns=NAME:.metadata.name,CLASS:.spec.storageClassName,RECLAIM:.spec.persistentVolumeReclaimPolicy
```

Provisioners differ in which class settings they honor. If your PVC bound immediately, or the PV shows `Delete`, your provisioner ignored those settings. Fix the PV with `kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'`.

**5. No default class: what happens?** (optional)

```bash
kubectl patch sc standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: no-default-claim
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
EOF
kubectl get pvc no-default-claim                     # Pending: no class and no default
kubectl delete pvc no-default-claim
kubectl patch sc standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**6. Optional: snapshots with minikube's CSI hostpath driver**

```bash
minikube addons enable volumesnapshots
minikube addons enable csi-hostpath-driver
kubectl get sc                                       # a CSI class, for example csi-hostpath-sc
kubectl get volumesnapshotclass                      # for example csi-hostpath-snapclass
```

Names can differ by minikube version. Create a PVC with that class, write a file, create a `VolumeSnapshot` (YAML above) using the snapshot class you see, then restore it into a new PVC with `dataSource` and check the file.

**Cleanup**

```bash
kubectl delete -f pod-custom.yaml -f pvc-custom.yaml -f storageclass-custom.yaml --ignore-not-found
kubectl delete -f pod-dynamic.yaml -f pvc-dynamic.yaml --ignore-not-found
kubectl get pv                                       # delete any leftover Retain volumes by hand
```

## Gotchas

- **`Pending` with `WaitForFirstConsumer` is normal**: create the Pod.
- **EBS is zonal.** Without delayed binding you can create a volume the Pod can never use.
- Dynamic volumes default to **`Delete`**: removing the PVC removes the disk. Use `Retain` (or snapshots) for important data.
- A class's `provisioner` and `parameters` are **immutable**. Create a new class instead.
- Two default classes, or none, produce confusing behavior for claims without a class.
- Not every driver supports expansion, snapshots or RWX. Check the driver's feature list.

## Check yourself

1. What does a StorageClass do?
2. What is the difference between static and dynamic provisioning?
3. Why use `WaitForFirstConsumer` with AWS EBS?
4. What happens to a PVC that omits the class if the cluster has no default class?
5. What is CSI, and which part mounts the volume on the node?
6. What do you set on the class to allow growing claims?
7. Which setting controls whether data survives deleting the claim?

<details>
<summary>Answers</summary>

1. It defines how to create storage on demand: the provisioner, its parameters, reclaim policy and binding mode
2. Static: an admin creates the PV first. Dynamic: the PV is created automatically when a PVC asks for a class
3. EBS volumes live in one zone. Waiting for the Pod's placement creates the volume in the right zone
4. It stays `Pending`
5. The Container Storage Interface, the standard for storage drivers. The CSI **node plugin** mounts it
6. `allowVolumeExpansion: true`
7. `reclaimPolicy` (`Retain` keeps it, `Delete` removes it)

</details>

---

[← Persistent Volume Claims](persistent-volume-claims.md) | [README](README.md)