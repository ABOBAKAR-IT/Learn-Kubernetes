# Module 06: Storage

> Keep data safe: volumes, PV, PVC and StorageClass.

**Notes ready:** 6/6 topics

## Topic directory

| Topic | File | Question it answers | Status |
|---|---|---|---|
| **Volumes** | [`volumes.md`](volumes.md) | Why does my data disappear, and how do volumes work? | 📘 |
| **emptyDir** | [`emptydir.md`](emptydir.md) | How do containers in a Pod share temporary files? | 📘 |
| **hostPath** | [`hostpath.md`](hostpath.md) | How do I use a folder from the node, and why is it risky? | 📘 |
| **Persistent Volumes** | [`persistent-volumes.md`](persistent-volumes.md) | What is the storage itself, and what is its lifecycle? | 📘 |
| **Persistent Volume Claims** | [`persistent-volume-claims.md`](persistent-volume-claims.md) | How does an app ask for storage? | 📘 |
| **Storage Classes** | [`storage-classes.md`](storage-classes.md) | Who creates volumes automatically? | 📘 |

YAML for the labs: [`labs/06-storage/manifests/`](../labs/06-storage/manifests/). One continuous walkthrough of the PV and PVC chain: [`labs/06-storage`](../labs/06-storage/).

## The big idea

> **The longer the data must live, the further from the container it must be stored.**

```
container filesystem   dies with the container
emptyDir               lives as long as the Pod                   (scratch, caches, sidecar sharing)
hostPath               lives on one node's disk                   (node agents, local learning)
PV / PVC               lives independently of Pods and nodes      (databases, uploads)
```

## The persistent trio

```
Pod ──volumes──► PVC ══binds══► PV ──► real storage (EBS, EFS, NFS, local disk)
                  ▲                ▲
              developer          admin, or a StorageClass creating it on demand
```

| Object | Role | Scope | Created by |
|---|---|---|---|
| **PV** | The storage | Cluster | Admin, or automatically |
| **PVC** | The request | Namespace | Developer |
| **StorageClass** | The recipe for creating PVs | Cluster | Admin |

### Static vs dynamic provisioning

```
Static:   admin creates PV ──► developer creates PVC ──► PVC binds the existing PV
Dynamic:  developer creates PVC (names a class) ──► provisioner creates the PV ──► they bind
```

## Which volume do I use?

| I need... | Use |
|---|---|
| Temporary scratch space or a cache | **emptyDir** |
| Containers in one Pod to share files | **emptyDir** |
| A node agent to read the node's files | **hostPath** (read-only, narrow path) |
| A database or uploads that must survive Pod deletion | **PVC** (dynamic, via a StorageClass) |
| One volume **per replica** | **StatefulSet** with `volumeClaimTemplates` |
| Shared read-write files across nodes | An **RWX** class (EFS, NFS) |
| Config files or credentials as files | **configMap / secret / projected** volumes |
| Fast local disks, scheduled onto the right node | A **local** PV with node affinity |

## Comparison

| | emptyDir | hostPath | PV / PVC |
|---|---|---|---|
| Survives container restart | Yes | Yes | Yes |
| Survives Pod deletion | **No** | Yes (same node) | **Yes** |
| Survives moving to another node | No | **No** | **Yes** (network storage) |
| Managed by | The Pod | The node | Kubernetes (PV lifecycle) |
| Good for production data | No | No | **Yes** |

## Recommended study order

| # | Topic | Why now |
|---|---|---|
| 1 | [Volumes](volumes.md) | The two-part model and every type |
| 2 | [emptyDir](emptydir.md) | The simplest volume |
| 3 | [hostPath](hostpath.md) | Node storage, and why it is not enough |
| 4 | [Persistent Volumes](persistent-volumes.md) | The storage and its lifecycle |
| 5 | [Persistent Volume Claims](persistent-volume-claims.md) | How apps request it |
| 6 | [Storage Classes](storage-classes.md) | How it is created on demand |

## Labs

- [labs/06-storage](../labs/06-storage/): static PV → PVC → Pod, dynamic provisioning, the `Released` state, with bugs found in the original files
- Each note has its own lab (projected volumes, shared emptyDir, eviction by size limit, PV protection, three `storageClassName` states, custom StorageClass)
- StatefulSet per-Pod volumes: [03 StatefulSets](../03-kubernetes-workloads/statefulsets.md)

## Module quiz

1. What happens to data in a container's own filesystem when it restarts?
2. Which two fields link a volume to a container mount?
3. Does an emptyDir survive a container restart? A Pod deletion?
4. Why is hostPath unsuitable for production data, and what is the safer alternative for node disks?
5. What is the difference between a PV, a PVC and a StorageClass?
6. Does a Pod reference a PV directly?
7. Name three rules for a PVC to bind to a PV.
8. What is the difference between omitting `storageClassName` and setting it to `""`?
9. What do `Retain` and `Delete` do, and which is the default for dynamic volumes?
10. A PV is `Released`. Why will a new PVC not bind to it, and what fixes that?
11. Why use `WaitForFirstConsumer` with AWS EBS?
12. What does `ReadWriteOnce` allow, and what does `ReadWriteMany` need?
13. A PVC stays `Pending`. Where do you look first, and what are three common causes?
14. What happens when you delete a PVC that a running Pod uses?
15. How do you give each replica its own volume?
16. How do you protect a dynamically provisioned volume from deletion?

<details>
<summary>Answers</summary>

1. It is lost, the container starts from the image
2. `volumes[].name` and `volumeMounts[].name`
3. Yes for a container restart. No for Pod deletion
4. It is tied to one node and exposes the node's filesystem. A `local` PV with node affinity
5. PV: the storage. PVC: the request for it. StorageClass: the recipe that creates PVs on demand
6. No, only through the PVC's `claimName`
7. Same class, supported access mode, PV size at least the request
8. Omitting uses the default class (often dynamic provisioning). `""` means no class (static PVs only)
9. `Retain` keeps the PV and data. `Delete` removes the PV and storage. `Delete` is the default for dynamic volumes
10. It still holds the old `claimRef`. Clear it with `kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'` after cleanup
11. EBS volumes live in one zone. Delaying creation until the Pod is scheduled puts the volume in the right zone
12. RWO: read-write by one node. RWX needs storage that supports it (EFS, NFS, CephFS)
13. `kubectl describe pvc <name>` (Events). Causes: no matching PV, class mismatch or missing default class, waiting for a consumer
14. It stays `Terminating` until the Pod stops using it (PVC protection)
15. A StatefulSet with `volumeClaimTemplates`
16. Patch the PV's `persistentVolumeReclaimPolicy` to `Retain` (and take snapshots or backups)

</details>

## Self-test before moving on

1. Draw the chain from Pod to real storage for static and for dynamic provisioning.
2. Delete a Pod that uses a PVC and prove the data survived.
3. Make a PV go `Released`, then make it `Available` again.
4. Create the three `storageClassName` variants and explain what each did.
5. Take the module quiz without looking at the answers.

## How to study this module

1. Read each note in order.
2. Type the YAML yourself, do not copy-paste.
3. Run the lab, then break it on purpose.
4. When you have studied and practiced a note, change its `Status` line to `✅ Studied`.

Next: [Module 08: Health and Scaling](../08-health-and-scaling/) in the recommended order, or [Module 07: Scheduling](../07-scheduling/)