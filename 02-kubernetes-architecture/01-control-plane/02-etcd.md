# etcd

> Status: 📘 Notes ready

## Idea

**etcd** is a distributed, consistent key-value database. It stores **all cluster state**: every Pod, Deployment, Service, ConfigMap, Secret, Node and RBAC rule.

```
Your YAML ─► API server ─► etcd   (the single source of truth)
```

- Only the **API server** reads or writes it.
- If etcd is lost and not backed up, the cluster definition is gone.
- Objects live under keys like `/registry/pods/default/web`.

## How it stays consistent: Raft and quorum

etcd runs as a cluster of members and uses the **Raft** protocol. One member is the **leader**, the others are **followers**. A write is committed only when a **majority (quorum)** agrees.

```
quorum = floor(members / 2) + 1
```

| Members | Quorum | Failures tolerated |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | **1** |
| 5 | 3 | **2** |
| 7 | 4 | 3 |

- Use **odd** numbers: 4 members tolerate no more failures than 3.
- Production control planes use **3 or 5** members, spread across failure zones.

## What lives in etcd

| Stored | Note |
|---|---|
| All API objects (desired state and status) | Pods, Deployments, Services... |
| **Secrets** | Only base64-encoded by default. Turn on **encryption at rest** |
| Leases, events | Events are short-lived |

## Performance and limits

- Needs **fast disk** (SSD) and low network latency. Slow disk means slow cluster.
- Default storage quota is about **2 GiB**. A full etcd goes read-only.
- Keep object sizes small. Do not store large blobs in ConfigMaps.

## Backup and restore (know this for production and the CKA)

```bash
# Snapshot (kubeadm-style paths; read the real ones from the etcd Pod, see the lab)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db

# Inspect it (newer versions use etcdutl)
etcdutl snapshot status /tmp/etcd-backup.db

# Restore into a new data directory
etcdutl snapshot restore /tmp/etcd-backup.db --data-dir=/var/lib/etcd-restored
```

Then point the etcd static Pod at the restored data directory.

## Lab

```bash
# 1. Find etcd (kubeadm, minikube and kind run it as a Pod on the control plane)
kubectl -n kube-system get pods -l component=etcd

# 2. Read the real certificate and endpoint paths from its definition
POD=$(kubectl -n kube-system get pod -l component=etcd -o name | head -1)
kubectl -n kube-system get $POD -o yaml | grep -E "cert-file|key-file|trusted-ca-file|listen-client"

# 3. List keys using etcdctl inside that Pod (replace the paths with those from step 2)
kubectl -n kube-system exec $POD -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  get /registry/namespaces --prefix --keys-only

# 4. Create something and find it in etcd
kubectl create namespace etcd-demo
# repeat step 3: /registry/namespaces/etcd-demo now appears
kubectl delete namespace etcd-demo
```

Managed clusters (EKS, GKE, AKS) do not give you access to etcd. The provider backs it up.

## Gotchas

- **Losing quorum** (for example 2 of 3 members down): the control plane cannot make changes. Running workloads usually keep running, but nothing can be scheduled or updated.
- Never edit etcd directly. Always go through the API server.
- Back up before **every** cluster upgrade.
- Secrets in etcd are readable by anyone with etcd access unless encryption at rest is enabled.

## Check yourself

1. What does etcd store?
2. How many failures can a 5-member etcd tolerate?
3. Why use an odd number of members?
4. What command takes a snapshot?
5. What happens to running Pods if etcd loses quorum?

<details>
<summary>Answers</summary>

1. The entire cluster state: all API objects including Secrets
2. Two (quorum is 3)
3. An even number adds a member without increasing the failures tolerated
4. `etcdctl snapshot save <file>` (with endpoints and certificates)
5. They keep running, but the cluster cannot be changed or healed until quorum returns

</details>