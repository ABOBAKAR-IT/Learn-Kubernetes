# kube-apiserver

> Status: 📘 Notes ready

## Idea

The **API server** is the front door of the cluster. Every request goes through it: `kubectl`, the scheduler, controllers, kubelets, your CI/CD.

- It exposes a **REST API** over HTTPS.
- It is the **only** component that reads and writes **etcd**.
- It is **stateless** (state lives in etcd), so you can run several copies behind a load balancer.

```
kubectl ─┐
kubelet ─┤
scheduler┼──► kube-apiserver ◄──► etcd
controllers┤
CI/CD ───┘
```

## What happens to every request

```
Request ─► 1. Authentication ─► 2. Authorization ─► 3. Admission ─► 4. Validation ─► 5. Save to etcd
            "who are you?"       "may you do this?"   "mutate / enforce"  "is the object valid?"
```

| Step | Question | Examples |
|---|---|---|
| **Authentication** | Who is this? | Client certificates, ServiceAccount tokens, OIDC, webhook |
| **Authorization** | Is it allowed to do this? | **RBAC** (default), Node authorizer, webhook |
| **Admission (mutating)** | Should the request be modified? | Add default StorageClass, inject sidecars, set defaults |
| **Admission (validating)** | Should it be rejected? | ResourceQuota, LimitRange, Pod Security, policy webhooks |
| **Persist** | Save the object | Written to etcd |

Failures return clear HTTP codes: `401` (not authenticated), `403` (forbidden), `422` (invalid object).

## The API structure

Resources are grouped into API groups and versions:

| Path | Group | Examples |
|---|---|---|
| `/api/v1` | Core | Pods, Services, ConfigMaps, Secrets, Nodes, Namespaces |
| `/apis/apps/v1` | `apps` | Deployments, ReplicaSets, DaemonSets, StatefulSets |
| `/apis/batch/v1` | `batch` | Jobs, CronJobs |
| `/apis/networking.k8s.io/v1` | `networking.k8s.io` | Ingress, NetworkPolicy |

This is the `apiVersion` field in your YAML: `v1` for core, `apps/v1` for the `apps` group.

A resource URL: `/api/v1/namespaces/default/pods/web`. Verbs: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`.

Versions mature as **alpha → beta → stable (v1)**. Alpha features are off by default.

## Watch: how everything stays in sync

Components do not poll. They open a **watch**, a long-lived HTTP stream, and the API server pushes every change.

```
scheduler: "tell me about Pods with no node"   ← watch
API server: "Pod web-1 was created"            ← pushed instantly
```

This is why the cluster reacts in seconds and why the API server can serve thousands of watchers.

## Lab

```bash
# 1. See the API calls kubectl makes
kubectl get pods -v=6                       # look for: GET https://.../api/v1/namespaces/default/pods

# 2. Discover what the API offers
kubectl api-versions
kubectl api-resources | head -20

# 3. Talk to the API directly
kubectl get --raw /readyz?verbose | head
kubectl get --raw /api/v1/namespaces/kube-system/pods | head -c 400

# 4. Plain HTTP via a local proxy (kubectl handles auth)
kubectl proxy --port=8001 &
curl -s http://localhost:8001/api/v1/namespaces/kube-system/pods | head -c 400
curl -s http://localhost:8001/apis/apps/v1/namespaces/kube-system/deployments | head -c 400
kill %1

# 5. Authorization
kubectl auth can-i create pods
kubectl auth can-i delete nodes
kubectl auth can-i list secrets --as=system:serviceaccount:default:default

# 6. Find the API server itself
kubectl -n kube-system get pod -l component=kube-apiserver -o wide
kubectl -n kube-system describe pod -l component=kube-apiserver | head -40
```

## Gotchas

- If the API server is down, **`kubectl` stops working but running Pods keep running** (kubelets keep their containers alive).
- Managed services (EKS, GKE, AKS) run the API server for you. You cannot SSH into it.
- A `403` is an RBAC problem, a `401` is a credentials problem.

## Check yourself

1. Which component is the only one that talks to etcd?
2. Name the order of steps a request passes through.
3. What does `apiVersion: apps/v1` mean?
4. Why do components use "watch" instead of polling?
5. What happens to running Pods if the API server goes down?

<details>
<summary>Answers</summary>

1. kube-apiserver
2. Authentication, authorization, admission (mutating then validating), validation, persist to etcd
3. The `apps` API group, version `v1` (served at `/apis/apps/v1`)
4. It is instant and efficient: the server pushes changes over a long-lived stream
5. They keep running; you just cannot change anything until it returns

</details>