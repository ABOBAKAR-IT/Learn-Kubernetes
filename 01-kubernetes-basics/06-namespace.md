# Namespaces

> Status: 📘 Notes ready

## Idea

A **Namespace** is a virtual folder inside one cluster. It groups objects so that teams, environments or apps do not collide.

```
Cluster
├── namespace: dev        → Deployment "web", Service "web", ConfigMap "app-config"
├── namespace: staging    → Deployment "web", Service "web", ConfigMap "app-config"
└── namespace: prod       → Deployment "web", Service "web", ConfigMap "app-config"
```

Same names, no conflict, because each lives in its own namespace.

## The four built-in namespaces

| Namespace | Purpose |
|---|---|
| `default` | Where objects go if you do not say otherwise |
| `kube-system` | Kubernetes' own components (CoreDNS, kube-proxy, scheduler...) |
| `kube-public` | Data readable by everyone (rarely used) |
| `kube-node-lease` | Node heartbeat Leases |

## Commands

```bash
kubectl get namespaces                     # or: kubectl get ns
kubectl create namespace dev
kubectl get pods -n dev
kubectl get pods -A                        # all namespaces
kubectl describe ns dev
kubectl config set-context --current --namespace=dev   # make dev your default
kubectl delete namespace dev               # deletes EVERYTHING inside it
```

As YAML:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    env: dev
```

Put an object in a namespace either with `-n dev` or in its metadata:

```yaml
metadata:
  name: web
  namespace: dev
```

## Namespaced vs cluster-scoped objects

Not everything lives in a namespace.

| Namespaced | Cluster-scoped |
|---|---|
| Pods, Deployments, ReplicaSets, Services | Nodes |
| ConfigMaps, Secrets | PersistentVolumes |
| PersistentVolumeClaims | Namespaces themselves |
| Roles, RoleBindings, ServiceAccounts | StorageClasses, ClusterRoles |

```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

## Talking across namespaces (DNS)

A Service name resolves differently depending on where you call from:

```
web-svc                            → same namespace only
web-svc.dev                        → from any namespace
web-svc.dev.svc.cluster.local      → fully qualified
```

## Limits per namespace

Two objects control resources inside a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 8Gi
---
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-defaults
  namespace: dev
spec:
  limits:
    - type: Container
      default:            # default limits
        cpu: 500m
        memory: 256Mi
      defaultRequest:     # default requests
        cpu: 100m
        memory: 128Mi
```

- **ResourceQuota**: a cap for the whole namespace.
- **LimitRange**: defaults and min/max per container.

Namespaces are also where you attach **RBAC** (who can do what) and **NetworkPolicies**.

## Lab

```bash
kubectl create namespace dev
kubectl create namespace prod

kubectl create deployment web --image=nginx:1.27 -n dev
kubectl create deployment web --image=nginx:1.27 --replicas=2 -n prod   # same name, no conflict

kubectl get deploy -A | grep web
kubectl get pods -n dev
kubectl get pods -n prod

kubectl config set-context --current --namespace=dev
kubectl get pods                         # now shows dev without -n

kubectl expose deployment web --port=80 -n dev
kubectl run tmp -n prod --rm -it --image=busybox:1.36 -- wget -qO- http://web.dev   # cross-namespace call

kubectl config set-context --current --namespace=default
kubectl delete namespace dev prod        # everything inside goes too
```

## Gotchas

- Deleting a namespace deletes **everything inside it**. Check the name twice.
- Namespaces are **not** a security or network boundary by default. A Pod in `dev` can still call a Pod in `prod` unless you add NetworkPolicies and RBAC.
- `kubectl get all -n dev` does not list ConfigMaps, Secrets, PVCs or Ingresses.
- A namespace stuck in `Terminating` usually has a resource with a finalizer that cannot finish. Inspect with `kubectl api-resources --verbs=list --namespaced -o name`.

## Check yourself

1. What problem do namespaces solve?
2. Name two cluster-scoped resources.
3. How do you call a Service in another namespace by DNS?
4. Does a namespace isolate network traffic by default?
5. How do you set a default namespace for your current context?

<details>
<summary>Answers</summary>

1. They group and separate objects so names, quotas and permissions do not collide
2. Nodes, PersistentVolumes, StorageClasses, ClusterRoles, Namespaces (any two)
3. `<service>.<namespace>` or `<service>.<namespace>.svc.cluster.local`
4. No. You need NetworkPolicies
5. `kubectl config set-context --current --namespace=<name>`

</details>