# DaemonSets

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/)

[← Deployments](deployments.md) | [README](README.md) | [StatefulSets →](statefulsets.md)

---

## Idea

A **DaemonSet** manages **node agents**. Like a ReplicaSet or Deployment it handles Pods and application updates, but with a different placement rule:

```
ReplicaSet / Deployment:  "run N Pods somewhere"        (no control over node placement)
DaemonSet:                "run exactly 1 Pod per Node"   (on all nodes, or a chosen subset)
```

**Typical uses:** monitoring and log collection on every node, storage daemons, networking and proxy daemons.
**Real examples:** `kube-proxy` runs as a DaemonSet Pod on every node. The Calico or Cilium node agents that implement Pod networking are DaemonSet Pods too.

## Behavior

| Event | Result |
|---|---|
| A node joins the cluster | A DaemonSet Pod is placed on it automatically |
| A node crashes or is removed | Its DaemonSet Pod is garbage collected |
| The DaemonSet is deleted | All Pods it created are deleted |

Course note: DaemonSet Pods are placed on nodes by the DaemonSet controller itself rather than the default scheduler. The default scheduler can take over this job if the corresponding feature is enabled. Placement is still limited by scheduling properties.

## Limit a DaemonSet to some nodes

| Tool | Effect |
|---|---|
| `nodeSelector` | Only nodes with a given label |
| Node affinity rules | Flexible label rules |
| Taints and tolerations | Allow Pods onto tainted nodes (for example control-plane nodes) |

## Picture

```
Node 1: [fluentd]     Node 2: [fluentd]     Node 3: [fluentd]  ← add Node 4 → [fluentd] appears
```

## YAML explained

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-agent
  namespace: default
  labels:
    k8s-app: fluentd-agent
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-agent
  template:
    metadata:
      labels:
        k8s-app: fluentd-agent
    spec:
      containers:
      - name: fluentd
        image: quay.io/fluentd_elasticsearch/fluentd:v4.5.2
```

Notice: **no `replicas` field**. The number of matching nodes decides the Pod count.

## Commands

```bash
kubectl apply -f fluentd-ds.yaml
kubectl get daemonsets            # or: kubectl get ds
kubectl get ds -o wide
kubectl get ds fluentd-agent -o yaml
kubectl describe ds fluentd-agent

# Updates and rollback work like Deployments
kubectl set image ds fluentd-agent fluentd=quay.io/fluentd_elasticsearch/fluentd:v4.6.2
kubectl rollout status  ds fluentd-agent
kubectl rollout history ds fluentd-agent
kubectl rollout history ds fluentd-agent --revision=2
kubectl rollout undo    ds fluentd-agent --to-revision=1

kubectl get all -l k8s-app=fluentd-agent -o wide
kubectl delete ds fluentd-agent
```

## Lab 5

> The manifests for this note are in [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/). Run the commands from that folder: `cd labs/02-deployment/manifests`.

A multi-node cluster makes this visible:

```bash
minikube start --nodes 3 -p multi          # or use kind with 3 nodes
kubectl get nodes
kubectl apply -f fluentd-ds.yaml
kubectl get pods -o wide -l k8s-app=fluentd-agent    # one Pod per worker node
kubectl get ds
kubectl delete ds fluentd-agent
```

Also look at real ones in your cluster:

```bash
kubectl get ds -n kube-system                        # kube-proxy, CNI agents
```

On a single-node minikube you will see just 1 Pod, which is correct.

## Gotchas

- Control-plane nodes are tainted. Add a toleration if you want DaemonSet Pods there.
- Updating works like a Deployment (rolling, with Revisions and rollback).
- Do not use a DaemonSet for ordinary apps. It is for per-node agents.

## Check yourself

1. How does a DaemonSet differ from a Deployment in Pod placement?
2. Give two real-world DaemonSet examples.
3. Why is there no `replicas` field?
4. What happens when a new node joins?

---

[← Deployments](deployments.md) | [README](README.md) | [StatefulSets →](statefulsets.md)