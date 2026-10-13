# Roles and ClusterRoles

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../../labs/08-rbac/)

[← RBAC overview](README.md) | [Module README](../README.md) | [Bindings →](rolebindings-and-clusterrolebindings.md)

---

## Idea

A **Role** (namespaced) or **ClusterRole** (cluster-wide) is a **list of rules**. Each rule says **which actions (verbs) are allowed on which resources**. A role grants nothing until a [binding](rolebindings-and-clusterrolebindings.md) gives it to someone.

## Role vs ClusterRole

| | Role | ClusterRole |
|---|---|---|
| Lives in | One namespace | The cluster (no namespace) |
| Can grant | Namespaced resources **in its own namespace** | Namespaced resources **in any namespace**, plus **cluster-scoped** resources, plus non-resource URLs |
| Typical use | "Read Pods in `dev`" | "Read Nodes", or a reusable set like `view` |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                       # or ClusterRole (then no metadata.namespace)
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

## Anatomy of a rule

```yaml
rules:
  - apiGroups: ["apps"]                       # 1) which API group
    resources: ["deployments", "deployments/scale"]   # 2) which resources (and subresources)
    resourceNames: ["web"]                    # 3) optional: only these named objects
    verbs: ["get", "patch"]                   # 4) which actions
```

A request is allowed if **any rule in any applicable role** matches **all four** parts.

### 1. `apiGroups`

The group is the part of `apiVersion` before the slash.

| `apiVersion` in YAML | `apiGroups` value |
|---|---|
| `v1` (Pods, Services, ConfigMaps, Secrets, Nodes...) | `""` (empty string, the **core** group) |
| `apps/v1` (Deployments, StatefulSets, DaemonSets, ReplicaSets) | `"apps"` |
| `batch/v1` (Jobs, CronJobs) | `"batch"` |
| `networking.k8s.io/v1` (Ingress, NetworkPolicy) | `"networking.k8s.io"` |
| `rbac.authorization.k8s.io/v1` | `"rbac.authorization.k8s.io"` |
| `autoscaling/v2` (HPA) | `"autoscaling"` |
| any | `"*"` |

Look them up instead of guessing:

```bash
kubectl api-resources -o wide                       # NAME, APIVERSION, NAMESPACED, VERBS
kubectl api-resources --api-group=apps
kubectl api-resources --namespaced=false            # cluster-scoped resources (need a ClusterRole)
```

### 2. `resources` and subresources

Resources are lowercase **plural** names: `pods`, `deployments`, `secrets`, `configmaps`, `services`.

**Subresources** are separate resources written `resource/sub`, and they need their own permission:

| Subresource | What it does |
|---|---|
| `pods/log` | Read logs (`kubectl logs`) |
| `pods/exec` | Run commands in a container (`kubectl exec`) |
| `pods/portforward` | `kubectl port-forward` |
| `deployments/scale` | `kubectl scale` |
| `pods/eviction` | Evict a Pod (`kubectl drain`) |
| `serviceaccounts/token` | Request a token for a ServiceAccount |

Permission on `pods` does **not** include `pods/log`.

### 3. `resourceNames`

Limit a rule to specific objects:

```yaml
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]          # only this ConfigMap
  verbs: ["get", "update"]
```

Caveat: it cannot restrict `list`, `watch` or `create` (the object has no name yet or the call covers many). It works for `get`, `update`, `patch`, `delete`.

### 4. `verbs`

| Verb | HTTP | What it means |
|---|---|---|
| `get` | GET one | Read **one** object by name |
| `list` | GET many | List objects (this also returns their full contents) |
| `watch` | GET stream | Stream changes |
| `create` | POST | Create |
| `update` | PUT | Replace |
| `patch` | PATCH | Modify part of it |
| `delete` | DELETE | Delete one |
| `deletecollection` | DELETE many | Delete many |

Special verbs for sensitive operations:

| Verb | Used for |
|---|---|
| `escalate` | Create or edit roles with **more** permissions than you hold |
| `bind` | Create bindings to roles you do not hold yourself |
| `impersonate` | Act as another user, group or ServiceAccount (`--as`) |
| `use` | Use a PodSecurityPolicy-style or similar resource |
| `approve`, `sign` | Certificate signing requests |

`*` means all verbs, which is almost always too much.

### Non-resource URLs (ClusterRole only)

```yaml
- nonResourceURLs: ["/healthz", "/livez", "/readyz", "/metrics"]
  verbs: ["get"]
```

## Which kubectl command needs which permission?

This table explains most "forbidden" errors:

| Command | Permissions it needs |
|---|---|
| `kubectl get pods` | `list` on `pods` |
| `kubectl get pod web` | `get` on `pods` |
| `kubectl get pods -w` | `list` and `watch` on `pods` |
| `kubectl describe pod web` | `get` on `pods`, and `list` on `events` |
| `kubectl logs web` | `get` on `pods/log` (and `get` on `pods`) |
| `kubectl exec -it web -- sh` | `get` on `pods` and **`create`** on `pods/exec` |
| `kubectl port-forward web 8080:80` | `get` on `pods` and `create` on `pods/portforward` |
| `kubectl delete pod web` | `delete` on `pods` |
| `kubectl apply -f deploy.yaml` | `get`, then `create` (new) or `patch` (existing) on `deployments` |
| `kubectl scale deploy web --replicas=3` | `get` on `deployments`, `patch` or `update` on **`deployments/scale`** |
| `kubectl rollout restart deploy web` | `get` and `patch` on `deployments` |
| `kubectl top pods` | `get`/`list` on `pods` in the `metrics.k8s.io` group |

## Examples

**Read-only access to Pods and their logs**

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

**Manage Deployments (including scaling)**

```yaml
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "deployments/scale"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

**Read nodes (cluster-scoped, so a ClusterRole)**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

**Update one named ConfigMap only**

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["feature-flags"]
    verbs: ["get", "update", "patch"]
```

**Several rules in one role**

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "patch"]
```

## Create them from the command line

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create role dep-manager --verb=get,list,create,patch --resource=deployments.apps -n dev
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create role cfg --verb=get,update --resource=configmaps --resource-name=feature-flags -n dev

# generate YAML for review or Git
kubectl create role pod-reader --verb=get,list --resource=pods -n dev --dry-run=client -o yaml
```

In the command line form, write grouped resources as `resource.group` (for example `deployments.apps`).

## Built-in ClusterRoles

```bash
kubectl get clusterroles | head -30
kubectl get clusterrole view -o yaml
kubectl describe clusterrole edit
```

| ClusterRole | Highlights |
|---|---|
| **`cluster-admin`** | `*` on `*`. Granted to the `system:masters` group. Full control, including RBAC itself |
| **`admin`** | Everything in a namespace including Roles and RoleBindings, but not the namespace's ResourceQuota or the Namespace object |
| **`edit`** | Create, change and delete most namespaced objects, **including Secrets**. Cannot change Roles or RoleBindings |
| **`view`** | Read most namespaced objects. **No Secrets**, no Roles or RoleBindings |
| `system:*` roles | For Kubernetes components (`system:node`, `system:kube-scheduler`...). Do not edit them |

`edit` can read Secrets and run Pods that mount them, so treat it as powerful.

## Aggregated ClusterRoles

An **aggregated** ClusterRole collects rules from other ClusterRoles by **label**. The built-in `admin`, `edit` and `view` work this way, so installing a CRD can extend them just by adding a labeled role.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring: "true"
rules: []                                   # filled in automatically
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
```

To extend the built-in `view` for a custom resource, label your ClusterRole `rbac.authorization.k8s.io/aggregate-to-view: "true"`.

## Gotchas

- **`list` returns full contents.** `list` (and `watch`) on `secrets` lets someone read **every secret value**, even without `get`.
- **Subresources are separate.** Reading logs needs `pods/log`, and `exec` needs `create` on `pods/exec`.
- **`apiGroups: [""]` is the core group.** A Deployment rule with `[""]` matches nothing. Use `["apps"]`.
- **Plural names:** `pod` and `deployment` do not match. Use `pods` and `deployments`.
- **`resourceNames` cannot limit `list` or `create`.**
- A **Role grants only inside its namespace**, even if it names a cluster-scoped resource (it silently grants nothing useful).
- Editing a role's `rules` takes effect immediately for every binding that references it.
- Wildcards (`*`) in verbs, resources or groups are a common source of over-permission.

## Check yourself

1. What is the difference between a Role and a ClusterRole?
2. What does `apiGroups: [""]` mean, and which group do Deployments belong to?
3. Which permissions does `kubectl exec` need?
4. Why is `list` on `secrets` dangerous?
5. What do `resourceNames` do, and what can they not restrict?
6. How does the built-in `view` role differ from `edit`?

<details>
<summary>Answers</summary>

1. A Role grants permissions inside one namespace. A ClusterRole can grant cluster-wide, cluster-scoped resources, non-resource URLs, and can be reused in any namespace through RoleBindings
2. The core API group (`v1` resources such as Pods and Secrets). Deployments are in the `apps` group
3. `get` on `pods` and `create` on `pods/exec`
4. `list` returns the full content of every Secret, so it exposes all the secret values
5. They limit a rule to named objects. They cannot restrict `list`, `watch` or `create`
6. `view` is read-only and excludes Secrets. `edit` can read and write most objects including Secrets

</details>

---

[← RBAC overview](README.md) | [Module README](../README.md) | [Bindings →](rolebindings-and-clusterrolebindings.md)
