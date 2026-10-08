# ReplicaSets

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/)

[← Pods](pods.md) | [README](README.md) | [Deployments →](deployments.md)

---

## Idea

A **ReplicaSet (RS)** is the next-generation ReplicationController. It implements **replication and self-healing**, with support for both selector types.

Why multiple replicas? One instance means one crash equals an outage. Running several instances in parallel gives **high availability**. The RS keeps the count correct, and you scale manually or with an autoscaler (HPA).

The replicas are **identical** (same image, same config) but still **distinct**: each has its own Pod name and IP, and the scheduler can place each on a different node.

## Picture: self-healing

```
Desired = 3

Current state matches:       A Pod dies:                RS reacts:
 [Pod1] [Pod2] [Pod3]         [Pod1] [ X  ] [Pod3]       [Pod1] [Pod4] [Pod3]
  3 = 3  ✔                     2 ≠ 3                      3 = 3  ✔
```

The loop that never stops:

```
count matching Pods ──► fewer than desired? create
                    └─► more than desired? delete
```

## YAML explained

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

| Field | Meaning |
|---|---|
| `replicas: 3` | Desired Pod count |
| `selector.matchLabels` | Which Pods the RS owns |
| `template` | Blueprint for new Pods (nested Pod: has `metadata` and `spec`, but no `apiVersion`/`kind`) |
| `template.metadata.labels` | **Must match** the selector |

## Commands

```bash
kubectl create -f redis-rs.yaml
kubectl apply  -f redis-rs.yaml
kubectl get replicasets            # or: kubectl get rs
kubectl scale rs frontend --replicas=4
kubectl get rs frontend -o yaml
kubectl get rs frontend -o json
kubectl describe rs frontend
kubectl delete rs frontend
```

## Lab 3

> The manifests for this note are in [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/). Run the commands from that folder: `cd labs/02-deployment/manifests`.

```bash
kubectl apply -f redis-rs.yaml
kubectl get rs,pods
kubectl delete pod <one-frontend-pod>
kubectl get pods                       # a replacement is already starting
kubectl scale rs frontend --replicas=5
kubectl get pods --show-labels
kubectl describe rs frontend           # read the Events (SuccessfulCreate)
kubectl delete rs frontend             # all its Pods go with it
```

## Gotchas

- An RS has limited features. It does not do rolling updates. That is the Deployment's job.
- You normally do **not** create an RS yourself. A Deployment creates it for you.
- If a bare Pod matches the RS selector, the RS can adopt it.

## Check yourself

1. How does the RS notice that a Pod died?
2. Name one cause for the "current state ≠ desired state" situation.
3. Why are three replicas "identical but distinct"?

---

## Legacy: ReplicationController (for recognition only)

### Idea

A **ReplicationController (RC)** ensures a **specified number of Pod replicas** run at any time, by constantly comparing actual state to desired state:

- more Pods than desired: it terminates the extra ones
- fewer Pods than desired: it requests more

It also supported application updates. It is **no longer recommended**. The default controller is the **Deployment**, which configures a **ReplicaSet**.

### YAML (for recognition only)

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: web-rc
spec:
  replicas: 3
  selector:
    app: web              # equality-based only, plain key/value (no matchLabels)
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.22.1
```

### Why it was replaced

| ReplicationController | ReplicaSet |
|---|---|
| Equality-based selectors only | Equality and set-based selectors |
| Plain `selector:` map | `selector.matchLabels` / `matchExpressions` |
| Replaced | Next generation, used by Deployments |

### Check yourself

1. Which controller replaced the RC?
2. What did the RC do when there were too many Pods?

---

[← Pods](pods.md) | [README](README.md) | [Deployments →](deployments.md)