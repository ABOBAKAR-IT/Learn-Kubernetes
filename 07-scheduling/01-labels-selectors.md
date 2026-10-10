# Labels and Selectors

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-scheduling/manifests/`](../labs/07-scheduling/manifests/)

[README](README.md) | [Node Selector →](node-selector.md)

---

## Idea

**Labels** are key/value tags attached to objects (Pods, Nodes, Deployments, Namespaces...). **Selectors** are queries over those tags. Together they are the **glue of Kubernetes**: nothing links by name, everything links by labels.

```
Deployment ──selector app=web──► finds its ReplicaSet and Pods
Service    ──selector app=web──► sends traffic to those Pods
Scheduler  ──nodeSelector disktype=ssd──► picks nodes
NetworkPolicy ──podSelector role=db──► protects those Pods
```

Labels are **not unique**: many objects can share a label, and one object can carry many labels.

## Rules for labels

| Part | Rule |
|---|---|
| **Key** | Optional prefix (a DNS name) + `/` + a name. The name is at most 63 characters of letters, digits, `-`, `_`, `.`, starting and ending with a letter or digit |
| **Value** | At most 63 characters, same characters, may be empty |
| Reserved prefixes | `kubernetes.io/` and `k8s.io/` belong to Kubernetes |

```yaml
metadata:
  labels:
    app: frontend
    env: qa
    team: payments
    app.kubernetes.io/name: shop
```

### Recommended labels

| Label | Example | Meaning |
|---|---|---|
| `app.kubernetes.io/name` | `shop` | The application |
| `app.kubernetes.io/instance` | `shop-prod` | This installation of it |
| `app.kubernetes.io/version` | `1.4.2` | Version |
| `app.kubernetes.io/component` | `api` | Role inside the app |
| `app.kubernetes.io/part-of` | `webstore` | The larger system |
| `app.kubernetes.io/managed-by` | `helm` | The tool managing it |

Using the same label names everywhere makes selectors, dashboards and cost reports consistent.

## Picture

Four Pods with two label keys, `app` and `env`:

```
Pod 1: app=frontend  env=dev        env=dev                   → Pod 1, Pod 3
Pod 2: app=frontend  env=qa         app=frontend              → Pod 1, Pod 2
Pod 3: app=db        env=dev        app=frontend,env=qa (AND) → Pod 2 only
Pod 4: app=db        env=qa
```

## Two selector types

| Type | Operators | Example | Used by |
|---|---|---|---|
| **Equality-based** | `=`, `==`, `!=` | `env=dev`, `env!=prod` | Services, ReplicationController, `kubectl -l` |
| **Set-based** | `in`, `notin`, key exists, key does not exist | `env in (dev,qa)`, `!app` | ReplicaSet, Deployment, DaemonSet, Job |

Several requirements separated by commas are **ANDed**. There is no OR across different keys.

### In YAML

A **Service** takes a plain map (equality, AND):

```yaml
spec:
  selector:
    app: frontend
    env: qa
```

A **Deployment** (and ReplicaSet, DaemonSet, Job) takes `matchLabels` and `matchExpressions`:

```yaml
spec:
  selector:
    matchLabels:
      app: frontend
    matchExpressions:
      - key: env
        operator: In               # In | NotIn | Exists | DoesNotExist
        values: [dev, qa]
```

## Commands

```bash
kubectl get pods --show-labels
kubectl get pods -L app,env                         # label values as columns
kubectl get pods -l env=dev
kubectl get pods -l app=frontend,env=qa             # AND
kubectl get pods -l 'env in (dev,qa)'
kubectl get pods -l 'env notin (prod)'
kubectl get pods -l '!team'                         # no "team" label at all
kubectl get nodes --show-labels

kubectl label pod web env=qa                        # add
kubectl label pod web env=prod --overwrite          # change
kubectl label pod web team-                         # remove
kubectl label pods -l app=frontend tier=web         # label many at once
kubectl label node worker1 disktype=ssd             # nodes have labels too
```

### Field selectors: filter by object fields, not labels

```bash
kubectl get pods --field-selector status.phase=Running
kubectl get pods -A --field-selector spec.nodeName=worker1
```

## Labels on nodes

Kubernetes sets some node labels itself, and you add your own:

| Label | Example |
|---|---|
| `kubernetes.io/hostname` | `worker1` |
| `kubernetes.io/os`, `kubernetes.io/arch` | `linux`, `amd64` |
| `topology.kubernetes.io/zone` | `eu-west-1a` |
| `topology.kubernetes.io/region` | `eu-west-1` |
| `node.kubernetes.io/instance-type` | `m5.large` |
| `node-role.kubernetes.io/control-plane` | (empty value) |

The rest of this module uses node and Pod labels to steer scheduling.

## Labels vs annotations

| | Labels | Annotations |
|---|---|---|
| Purpose | **Identify and select** | **Describe**: extra, non-identifying metadata |
| Used in selectors | Yes | **No** |
| Size | Short (63 characters) | Large (up to about 256 KB in total) |
| Examples | `app=web`, `env=prod` | build URL, git commit, `kubernetes.io/change-cause`, tool settings |

## Lab

```bash
kubectl run p1 --image=nginx:1.27 --labels="app=frontend,env=dev"
kubectl run p2 --image=nginx:1.27 --labels="app=frontend,env=qa"
kubectl run p3 --image=nginx:1.27 --labels="app=db,env=dev"
kubectl run p4 --image=nginx:1.27 --labels="app=db,env=qa"
```

**1. Select**

```bash
kubectl get pods --show-labels
kubectl get pods -l env=dev                         # p1, p3
kubectl get pods -l app=frontend,env=qa             # p2
kubectl get pods -l 'env in (dev,qa),app=db'        # p3, p4
kubectl get pods -L app,env
```

**2. Change labels**

```bash
kubectl label pod p1 env=qa --overwrite
kubectl get pods -l env=qa                          # p1, p2, p4
kubectl label pod p1 env-                           # remove it
kubectl get pods -l '!env'                          # p1
```

**3. A Service follows labels**

```bash
kubectl expose pod p1 --port=80 --name=fe --selector=app=frontend
kubectl get endpointslices -l kubernetes.io/service-name=fe       # p1 and p2
kubectl label pod p2 app=other --overwrite
kubectl get endpointslices -l kubernetes.io/service-name=fe       # only p1 now
```

**4. Orphan a Pod from its Deployment**

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl get pods -l app=web
POD=$(kubectl get pod -l app=web -o name | head -1)
kubectl label $POD app-                             # remove the label that links it
kubectl get pods -l app=web                         # a replacement was created: still 3
kubectl get pods                                    # the orphan is still running, now unmanaged
kubectl delete $POD
```

**5. Node labels**

```bash
kubectl get nodes --show-labels
kubectl get nodes -L kubernetes.io/arch,kubernetes.io/os
```

**Cleanup**

```bash
kubectl delete pod p1 p2 p3 p4
kubectl delete svc fe
kubectl delete deployment web
```

## Gotchas

- A **typo in a label** silently breaks a Service, a Deployment or a NetworkPolicy. No error, just nothing matching.
- A Deployment's `spec.selector` is **immutable** after creation, and it must match `template.metadata.labels`.
- Two Deployments with **overlapping selectors** can fight over Pods. Keep selectors unique, for example by including `app.kubernetes.io/instance`.
- Changing a Pod's label can **detach** it from a controller or a Service. This is a handy debugging trick (it keeps the Pod for inspection).
- Label **values are strings**: quote numbers like `"1"` and booleans like `"true"`.

## Check yourself

1. What is the difference between a label and an annotation?
2. Which selector type do Services use, and which do Deployments use?
3. How do you select Pods that have `env` of `dev` or `qa` and also `app=db`?
4. What happens if you remove the controlling label from a Pod owned by a ReplicaSet?
5. Name three labels Kubernetes puts on nodes.

<details>
<summary>Answers</summary>

1. Labels identify and are used in selectors. Annotations hold larger, non-identifying metadata and cannot be selected on
2. Services: an equality-based map. Deployments: `matchLabels` and `matchExpressions` (set-based)
3. `kubectl get pods -l 'env in (dev,qa),app=db'`
4. The ReplicaSet no longer counts it and creates a replacement. The old Pod keeps running, unmanaged
5. `kubernetes.io/hostname`, `kubernetes.io/os`, `kubernetes.io/arch`, `topology.kubernetes.io/zone` (any three)

</details>

---

[README](README.md) | [Node Selector →](node-selector.md)