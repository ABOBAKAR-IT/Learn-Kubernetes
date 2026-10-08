# YAML Manifests

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/04-configmap-secret/manifests/`](../labs/04-configmap-secret/manifests/)

[README](README.md) | [ConfigMaps →](configmaps.md)

---

## Idea

A **manifest** is a YAML (or JSON) file that describes Kubernetes objects. It is the **desired state**, and when you keep manifests in Git, Git becomes the source of truth for your cluster.

Learn this first in the module: every ConfigMap, Secret and Deployment you write is a manifest, and most beginner errors are YAML mistakes.

## YAML in 5 minutes

```yaml
# 1. Maps (key: value). Indent with 2 spaces. NEVER use tabs.
metadata:
  name: web
  labels:
    app: web

# 2. Lists: each item starts with "- "
containers:
  - name: nginx
    image: nginx:1.27
  - name: sidecar
    image: busybox:1.36

# 3. Strings: quote when in doubt
value: "8080"          # a string
port: 8080             # a number
enabled: "true"        # a string (env values MUST be strings)
enabled: true          # a boolean

# 4. Multi-line text
script: |              # keeps line breaks (use for config files)
  line one
  line two
text: >                # folds lines into one paragraph
  this becomes
  one line

# 5. Empty map / list
emptyDir: {}
args: []
```

| Rule | Why it matters |
|---|---|
| **Space after the colon** | `containerPort:8080` is one string, `containerPort: 8080` is a key and value |
| **Spaces only, 2 per level** | A tab is a syntax error |
| **List items align** | `- name` items under the same key must line up |
| **Quote ambiguous values** | `yes`, `no`, `on`, `off`, `true`, `1.20` can be read as booleans or numbers |

## Anatomy of every object

```yaml
apiVersion: apps/v1        # API group and version (v1 for core objects)
kind: Deployment           # object type, case sensitive
metadata:                  # identity
  name: web
  namespace: dev
  labels:
    app.kubernetes.io/name: web
  annotations:
    kubernetes.io/change-cause: "first release"
spec:                      # desired state (shape depends on kind)
  replicas: 3
# status:                  # written by Kubernetes, never by you
```

Recommended labels (shared vocabulary used by tools):

| Label | Example |
|---|---|
| `app.kubernetes.io/name` | `web` |
| `app.kubernetes.io/instance` | `web-prod` |
| `app.kubernetes.io/version` | `1.4.2` |
| `app.kubernetes.io/component` | `frontend` |
| `app.kubernetes.io/part-of` | `shop` |
| `app.kubernetes.io/managed-by` | `helm` |

## Several objects in one file

Separate objects with `---`. Put dependencies first (Namespace, ConfigMap, Secret, then the Deployment and Service):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
# ...
```

```bash
kubectl apply -f app.yaml            # one file, many objects
kubectl apply -f ./manifests/        # every file in a folder
kubectl apply -R -f ./k8s/           # folders inside folders
kubectl apply -k ./overlays/dev      # a Kustomize directory
```

## Finding the right fields

```bash
kubectl api-resources                              # kinds, short names, apiVersion group
kubectl api-versions
kubectl explain deployment.spec.strategy           # field documentation in the terminal
kubectl explain pod.spec.containers.envFrom --recursive
```

## Generating and cleaning YAML

```bash
# Generate a starter file without creating anything
kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Export a live object (then clean it)
kubectl get deployment web -o yaml > web.yaml
```

When you export a live object, delete these fields before reusing it: `status`, `metadata.uid`, `metadata.resourceVersion`, `metadata.creationTimestamp`, `metadata.managedFields`, and the `kubectl.kubernetes.io/last-applied-configuration` annotation.

## Validating before you apply

| Tool | Checks | Needs cluster |
|---|---|---|
| `yamllint file.yaml` | YAML syntax and style | No |
| `kubectl apply -f f.yaml --dry-run=client` | Parses, builds the object locally | Light |
| `kubectl apply -f f.yaml --dry-run=server` | Full API validation (schema, admission) without saving | Yes |
| `kubectl diff -f f.yaml` | Shows what would change on the live object | Yes |
| `kubeconform -strict f.yaml` | Schema validation offline (good in CI) | No |

A good habit: **`diff` before `apply`**.

```bash
kubectl diff -f deploy.yaml
kubectl apply -f deploy.yaml
```

## Apply, replace and patch

| Command | Behavior |
|---|---|
| `kubectl apply -f` | Create or update; computes a three-way merge. Preferred |
| `kubectl apply --server-side -f` | Server tracks which manager owns each field (good with several tools) |
| `kubectl replace -f` | Replaces the whole object (must exist) |
| `kubectl create -f` | Create only, fails if it exists |
| `kubectl patch` | Change a few fields without a file |

```bash
kubectl patch deployment web -p '{"spec":{"replicas":5}}'                      # strategic merge (default)
kubectl patch configmap app-config --type merge -p '{"data":{"LOG_LEVEL":"debug"}}'
kubectl patch deployment web --type json -p '[{"op":"replace","path":"/spec/replicas","value":2}]'
```

## Organizing manifests in a repo

```
k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.example.yaml        # placeholders only, real values never in Git
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

| Practice | Reason |
|---|---|
| One resource per file, named `<kind>.yaml` (or `<app>-<kind>.yaml`) | Easy diffs and reviews |
| Keep manifests next to the code or in a dedicated config repo | Versioned together |
| Never commit real Secrets | Use Sealed Secrets, SOPS or External Secrets (see [Secrets](secrets.md)) |
| Pin image tags (`nginx:1.27`), avoid `latest` | Repeatable deployments |
| Use `kubectl diff` and `--dry-run=server` in CI | Catch errors before the cluster does |
| Use Kustomize or Helm once you have several environments | Avoid copy-paste |

## Common errors and what they mean

| Error message (shortened) | Cause | Fix |
|---|---|---|
| `mapping values are not allowed in this context` | Missing or extra space/colon | Check indentation and `key: value` spacing |
| `found character that cannot start any token` | A tab character | Replace tabs with spaces |
| `no matches for kind "service"` | `kind` is case sensitive | `kind: Service` |
| `unknown field "spec.volume"` | Misspelled field | `volumes` (plural). Use `kubectl explain` |
| `cannot unmarshal string into ... ports` | `containerPort:8080` without a space | `containerPort: 8080` |
| `cannot unmarshal bool into ... value of type string` | `value: true` in an env var | `value: "true"` |
| `selector does not match template labels` | Deployment `selector` and `template.labels` differ | Make them identical |
| `spec.selector: Required value` | Missing `selector` in apps/v1 | Add `selector.matchLabels` |

Real mistakes from this repo's earlier labs: `kind: service`, `volume:` instead of `volumes:`, `containerPort:8080`, and `["sh","-c","sleep","3600"]` (the command after `-c` must be **one** string).

## Lab: break it, then fix it

```bash
cd labs/04-configmap-secret/manifests
cat broken-deployment.yaml                         # 4 mistakes are hidden in it
kubectl apply -f broken-deployment.yaml --dry-run=server
```

Fix one error at a time and re-run the command until it passes. Then compare with `fixed-deployment.yaml`:

```bash
diff broken-deployment.yaml fixed-deployment.yaml
kubectl apply -f fixed-deployment.yaml
kubectl get deploy broken-demo
kubectl delete -f fixed-deployment.yaml
```

Then practice the review loop:

```bash
kubectl create deployment web --image=nginx:1.27 --dry-run=client -o yaml > web.yaml
kubectl apply -f web.yaml
sed -i 's/nginx:1.27/nginx:1.26/' web.yaml
kubectl diff -f web.yaml                           # shows the image change
kubectl apply -f web.yaml
kubectl delete -f web.yaml
```

<details>
<summary>The four mistakes in broken-deployment.yaml</summary>

1. `kind: deployment` must be `Deployment`
2. `containerPort:80` needs a space: `containerPort: 80`
3. `value: true` must be a string: `value: "true"`
4. `volume:` must be `volumes:`

</details>

## Gotchas

- Indentation errors often show up as errors **far from the real mistake**. Check the lines above.
- Copying YAML from web pages can bring tabs or smart quotes. Use `yamllint`.
- A manifest that applies fine can still be wrong (for example the wrong port), because validation checks shape, not meaning.
- `kubectl apply` on an object created with `create` can warn about a missing annotation. Pick `apply` from the start.

## Check yourself

1. Why is `containerPort:8080` wrong?
2. What does `---` do in a manifest file?
3. How do you validate a manifest against the real API without creating anything?
4. Which command shows what would change before you apply?
5. Name four fields to remove when reusing exported YAML.

<details>
<summary>Answers</summary>

1. There is no space after the colon, so YAML reads it as one string instead of a key and value
2. Separates several objects in one file
3. `kubectl apply -f file.yaml --dry-run=server`
4. `kubectl diff -f file.yaml`
5. `status`, `uid`, `resourceVersion`, `creationTimestamp` (also `managedFields`)

</details>

---

[README](README.md) | [ConfigMaps →](configmaps.md)