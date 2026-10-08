# Deployments

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/)

[← ReplicaSets](replicasets.md) | [README](README.md) | [DaemonSets →](daemonsets.md)

---

## Idea

A **Deployment** gives **declarative updates** to Pods and ReplicaSets. The **DeploymentController** runs inside the control plane's controller manager and keeps current state equal to desired state.

It provides:

- **Rolling updates** (the default strategy): seamless version changes
- **Rollbacks**: go back to an earlier Revision
- **Scaling**: through its ReplicaSets
- **Recreate** strategy: a disruptive, less common alternative (kill all, then start all)

```
Deployment  ──creates──►  ReplicaSet  ──creates──►  Pods
```

You manage only the Deployment. It manages ReplicaSets and Pods on your behalf.

## YAML explained

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.20.2
        ports:
        - containerPort: 80
```

| Field | Meaning |
|---|---|
| `apiVersion: apps/v1` | The `apps` API group, which hosts Deployment, ReplicaSet, DaemonSet |
| `kind: Deployment` | Object type |
| `metadata` | Name, labels, annotations, namespace |
| `spec.replicas: 3` | Run 3 Pod instances at all times |
| `spec.selector.matchLabels` | Pods this Deployment owns |
| `spec.template` | Pod blueprint; a nested object **keeps** `metadata` and `spec`, and **loses** `apiVersion` and `kind` |
| `spec.template.spec` | The actual PodSpec (containers, image, ports) |

There are two `spec` fields: the Deployment's (`spec`) and the Pod's (`spec.template.spec`). Once created, Kubernetes adds a `status` field automatically.

## Imperative way and YAML generator

```bash
kubectl create deployment nginx-deployment \
  --image=nginx:1.20.2 --port=80 --replicas=3

# Generate the manifest instead of running it
kubectl create deployment nginx-deployment \
  --image=nginx:1.20.2 --port=80 --replicas=3 \
  --dry-run=client -o yaml > nginx-deploy.yaml
```

## Rolling update: how it works

A **Revision** is a recorded state of the Deployment. Each Revision is tied to one ReplicaSet.

```
Step 1: Create Deployment (image 1.20.2)
        Deployment ─► ReplicaSet A (1.20.2) ─► Pod Pod Pod        = Revision 1

Step 2: Change image to 1.21.5  (Pod template changed → rolling update)
        ReplicaSet B (1.21.5) is created. Pods move over gradually:
        A: ■ ■ ■   →   ■ ■   →   ■   →   (0)
        B:          →   □     →   □ □  →   □ □ □                 = Revision 2

Step 3: Finished
        Deployment ─► ReplicaSet B (1.21.5) ─► Pod Pod Pod   (active)
                      ReplicaSet A (1.20.2) ─► scaled to 0   (kept for rollback)
```

**What triggers a new Revision?** Changes to the **Pod template**: image, container port, volumes, mounts.

**What does NOT trigger one?** Dynamic operations: **scaling** and **labeling the Deployment**. These do not change the Revision number.

## Rollback

Because old ReplicaSets are kept (scaled to 0), rollback is just "scale the old one up, the new one down".

```
Revision 2 (nginx:1.21.5) performs badly
        │  kubectl rollout undo --to-revision=1
        ▼
Revision 1 (nginx:1.20.2) running again
```

## Commands (from the course, grouped)

```bash
# Create and inspect
kubectl apply -f nginx-deploy.yaml
kubectl get deployments
kubectl get deploy -o wide
kubectl get deploy nginx-deployment -o yaml
kubectl describe deploy nginx-deployment

# Scale (no new Revision)
kubectl scale deploy nginx-deployment --replicas=4

# Update the image (new Revision)
kubectl set image deploy nginx-deployment nginx=nginx:1.21.5

# Track the rollout
kubectl rollout status  deploy nginx-deployment
kubectl rollout history deploy nginx-deployment
kubectl rollout history deploy nginx-deployment --revision=1
kubectl rollout history deploy nginx-deployment --revision=2

# Roll back
kubectl rollout undo deploy nginx-deployment --to-revision=1

# See everything this Deployment created
kubectl get all -l app=nginx-deployment -o wide
kubectl get deploy,rs,po -l app=nginx-deployment

# Cleanup
kubectl delete deploy nginx-deployment
```

## Lab 4: rolling update and rollback

> The manifests for this note are in [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/). Run the commands from that folder: `cd labs/02-deployment/manifests`.

```bash
kubectl apply -f nginx-deploy.yaml
kubectl annotate deploy nginx-deployment kubernetes.io/change-cause="initial nginx 1.20.2"
kubectl get deploy,rs,po -l app=nginx-deployment     # 1 RS, 3 Pods

kubectl set image deploy nginx-deployment nginx=nginx:1.21.5
kubectl annotate deploy nginx-deployment kubernetes.io/change-cause="upgrade to 1.21.5" --overwrite
kubectl rollout status deploy nginx-deployment
kubectl get rs                                        # RS A = 0 Pods, RS B = 3 Pods

kubectl rollout history deploy nginx-deployment       # Revisions 1 and 2
kubectl rollout history deploy nginx-deployment --revision=2   # shows image 1.21.5

kubectl scale deploy nginx-deployment --replicas=4
kubectl rollout history deploy nginx-deployment       # still only 2 Revisions: scaling is not a new one

kubectl rollout undo deploy nginx-deployment --to-revision=1
kubectl describe deploy nginx-deployment | grep -i image   # back on 1.20.2
```

Break-it experiment:

```bash
kubectl set image deploy nginx-deployment nginx=nginx:does-not-exist
kubectl get pods                       # new Pod: ImagePullBackOff; old Pods still serve
kubectl rollout undo deploy nginx-deployment
```

## Gotchas

- Changing only `replicas` or labels on the Deployment does **not** roll out anything.
- `template.metadata.labels` must match `spec.selector.matchLabels`.
- `--record` is **deprecated** in modern kubectl. Use the `kubernetes.io/change-cause` annotation (shown above) to describe each Revision.

## Check yourself

1. Which objects does a Deployment create, and in which order?
2. What is a Revision, and how does it map to a ReplicaSet?
3. Does scaling a Deployment create a new Revision?
4. Why does the old ReplicaSet stay around with 0 Pods?

---

## Course commands that need a fix

| Course command | Problem | Use instead |
|---|---|---|
| `kubectl get all -l app=nginx -o wide` | The Deployment YAML labels Pods `app: nginx-deployment`, so `app=nginx` matches **nothing** | `kubectl get all -l app=nginx-deployment -o wide` |
| `kubectl get deploy,rs,po -l app=nginx` | Same label mismatch | `-l app=nginx-deployment` |
| `kubectl apply ... --record` and `kubectl set image ... --record` | `--record` is deprecated in modern kubectl | Annotate: `kubectl annotate deploy <name> kubernetes.io/change-cause="why" --overwrite` |
| `kubectl create -f` then later `kubectl apply -f` on the same object | Mixing them can warn about a missing last-applied annotation | Pick `apply` from the start |

Lesson hiding in the first row: **a wrong label selector returns nothing, with no error.** This is exactly how a Service ends up with no endpoints.

---

[← ReplicaSets](replicasets.md) | [README](README.md) | [DaemonSets →](daemonsets.md)