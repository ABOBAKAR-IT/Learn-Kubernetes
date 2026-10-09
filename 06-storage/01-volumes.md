# Volumes

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/06-storage/manifests/`](../labs/06-storage/manifests/) · Full walkthrough: [`labs/06-storage`](../labs/06-storage/)

[README](README.md) | [emptyDir →](emptydir.md)

---

## Idea

A container's own filesystem is **temporary**: it is created from the image and thrown away when the container is replaced. A **volume** is a directory that lives **outside** the container's filesystem and is mounted into it, so data (and sharing between containers) can survive.

| Event | Data in the container filesystem | Data in a volume |
|---|---|---|
| Container crashes and restarts | **Lost** | Kept (for every volume type) |
| Pod is deleted or rescheduled | **Lost** | Depends on the **type** |
| Node is lost | **Lost** | Only network or cloud storage survives |

> **The longer the data must live, the further from the container it must be stored.**

```
container filesystem   →  dies with the container
emptyDir               →  lives as long as the Pod
hostPath               →  lives on one node's disk
PersistentVolume       →  lives independently of Pods and (with network storage) nodes
```

## The two parts of every volume

```yaml
spec:
  volumes:                       # 1) DECLARE what exists (Pod level)
    - name: my-vol
      emptyDir: {}
  containers:
    - name: app
      volumeMounts:              # 2) MOUNT it into a container (container level)
        - name: my-vol           #    the name is the link
          mountPath: /data
```

The **`name`** connects the two. Declare without mounting (or the reverse) and the Pod fails to start. Several containers can mount the same volume, each at its own path.

### Mount options

| Field | Meaning |
|---|---|
| `mountPath` | Where the volume appears inside the container |
| `readOnly: true` | Mount read-only (good default for config and secrets) |
| `subPath` | Mount only one sub-directory or file of the volume |
| `subPathExpr` | Like `subPath`, with env var expansion, for example `$(POD_NAME)` |

`subPath` lets one volume serve several mounts:

```yaml
volumeMounts:
  - { name: data, mountPath: /var/log/app, subPath: logs }
  - { name: data, mountPath: /var/data,    subPath: data }
```

## Volume types

| Type | Lifetime | Typical use |
|---|---|---|
| [`emptyDir`](emptydir.md) | The Pod | Scratch space, caches, sharing between containers |
| [`hostPath`](hostpath.md) | The node | Node agents, local learning |
| [`persistentVolumeClaim`](persistent-volume-claims.md) | Independent | Databases, uploads |
| `ephemeral` (generic ephemeral volume) | The Pod | Per-Pod scratch space on real storage |
| `configMap` / `secret` | The object | Config files and credentials |
| `downwardAPI` | The Pod | Expose the Pod's own labels, name, limits as files |
| `projected` | The Pod | Combine several sources in one directory |
| `nfs`, CSI drivers | Remote | Shared or cloud storage |

### Projected volumes

A **projected** volume merges several sources into **one directory**:

```yaml
volumes:
  - name: all-in-one
    projected:
      sources:
        - configMap:
            name: app-settings
            items: [{ key: app.properties, path: app.properties }]
        - secret:
            name: app-creds
            items: [{ key: password, path: db-password }]
        - downwardAPI:
            items:
              - { path: podname, fieldRef: { fieldPath: metadata.name } }
              - { path: labels,  fieldRef: { fieldPath: metadata.labels } }
```

### Generic ephemeral volumes

A Pod can ask for a **per-Pod PVC** that is created with the Pod and deleted with it. Use it for large scratch space on real storage instead of node disk:

```yaml
volumes:
  - name: scratch
    ephemeral:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi
```

## Permissions

Volumes are often **root-owned**, and a container running as a normal user may not be able to write. Use the Pod `securityContext`:

```yaml
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000                 # volume files get group 2000 and group-write
    fsGroupChangePolicy: OnRootMismatch   # skip a slow recursive chown when already correct
```

## Ephemeral storage limits

The container's writable layer, `emptyDir` volumes and logs all count as **ephemeral storage** on the node. You can request and limit it:

```yaml
resources:
  requests:
    ephemeral-storage: 1Gi
  limits:
    ephemeral-storage: 2Gi
```

A Pod that exceeds its limit is **evicted**.

## Lab

```bash
cd labs/06-storage/manifests
```

**1. Projected volume: several sources, one directory**

```bash
kubectl apply -f projected-volume-demo.yaml
kubectl exec projected-demo -- ls -l /etc/projected
kubectl exec projected-demo -- cat /etc/projected/app.properties
kubectl exec projected-demo -- cat /etc/projected/db-password
kubectl exec projected-demo -- cat /etc/projected/podname
kubectl exec projected-demo -- cat /etc/projected/labels

# change the ConfigMap and watch the mounted file follow (can take about a minute)
kubectl patch configmap app-settings -p '{"data":{"app.properties":"server.port=9090\n"}}'
kubectl exec projected-demo -- cat /etc/projected/app.properties
```

**2. subPath: one volume, two mount points**

```bash
kubectl apply -f subpath-demo.yaml
kubectl exec subpath-demo -c writer -- sh -c 'touch /var/log/app/a.log /var/data/b.dat'
kubectl exec subpath-demo -c viewer -- ls -R /all        # sees logs/a.log and data/b.dat
```

**3. Permissions with `fsGroup`**

```bash
kubectl apply -f volume-permissions.yaml
kubectl exec perm-demo -- id                              # uid 1000, groups include 2000
kubectl exec perm-demo -- ls -ld /data                    # group 2000 and the setgid "s" bit
kubectl exec perm-demo -- sh -c 'echo ok > /data/test && ls -l /data'
```

Break it: remove `fsGroup` from the YAML, recreate the Pod, and compare `ls -ld /data`.

**Cleanup**

```bash
kubectl delete -f projected-volume-demo.yaml -f subpath-demo.yaml -f volume-permissions.yaml
```

## Gotchas

- A volume that is **declared but not mounted** (or the reverse, a mount with no matching volume) is an error or silently unused.
- A mount **hides** whatever the image had at that path.
- A ConfigMap or Secret mounted with **`subPath` never updates**.
- Containers in one Pod can **share** a volume, which is how sidecars exchange files.
- `fsGroup` can make mounts slow on large volumes, since files may be re-owned at every start (use `fsGroupChangePolicy`).

## Check yourself

1. What happens to a container's own filesystem when it restarts?
2. Which two fields link a volume to a container mount?
3. Name three volume types and how long each one lives.
4. What does `subPath` do, and what is its main gotcha?
5. A container running as UID 1000 cannot write to a volume. What do you set?

<details>
<summary>Answers</summary>

1. It is lost: the container starts from the image's fresh filesystem
2. `volumes[].name` and `volumeMounts[].name` (they must match)
3. Example: emptyDir (the Pod), hostPath (the node), PVC (independent of Pods)
4. It mounts one sub-directory or file of the volume. ConfigMap/Secret mounts using it do not update
5. `securityContext.fsGroup` (and `runAsUser`) on the Pod

</details>

---

[README](README.md) | [emptyDir →](emptydir.md)