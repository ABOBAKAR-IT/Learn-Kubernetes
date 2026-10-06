# 🍕 Kubernetes ConfigMap & Secret – Beginner Friendly Guide

Learn how to **separate configuration from your application** using Kubernetes `ConfigMap` and `Secret`, with a simple hands-on pizza example.

---

## 📚 Table of Contents

1. [Why do we need ConfigMap and Secret?](#-why-do-we-need-configmap-and-secret)
2. [ConfigMap vs Secret](#-configmap-vs-secret)
3. [Project Structure](#-project-structure)
4. [Prerequisites](#-prerequisites)
5. [Step-by-Step Walkthrough](#-step-by-step-walkthrough)
6. [Understanding Each File](#-understanding-each-file)
7. [Verify Everything](#-verify-everything)
8. [Other Ways to Use ConfigMap and Secret](#-other-ways-to-use-configmap-and-secret)
9. [Common Mistakes](#-common-mistakes)
10. [Security Notes](#-security-notes)
11. [Cheat Sheet](#-cheat-sheet)
12. [Cleanup](#-cleanup)

---

## 🤔 Why do we need ConfigMap and Secret?

Imagine your app needs these settings:

- `PIZZA_TYPE = Cheese`
- `PAYMENT_MODE = Cash`
- `DB_PASSWORD = superuser`

You **could** hardcode them inside the container image, but then:

- ❌ You must rebuild the image for every change
- ❌ Dev, test and prod need different images
- ❌ Passwords end up inside the image (dangerous!)

**Solution:** keep configuration *outside* the image and inject it when the Pod starts.

| Kubernetes object | Used for |
|---|---|
| **ConfigMap** | Non-sensitive config (URLs, modes, feature flags) |
| **Secret** | Sensitive data (passwords, tokens, API keys) |

---

## ⚖️ ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---|---|---|
| Purpose | Normal configuration | Sensitive data |
| Value format in YAML | Plain text | Base64 encoded (`data`) or plain (`stringData`) |
| Typical content | `PIZZA_TYPE=Cheese` | `DB_PASSWORD=superuser` |
| Size limit | 1 MiB | 1 MiB |
| Extra protections | None | Can be encrypted at rest, RBAC-restricted, mounted in memory (tmpfs) |

> ⚠️ **Base64 is NOT encryption.** It is just encoding. Anyone can decode it. See [Security Notes](#-security-notes).

---

## 📁 Project Structure

```
k8s-configmap-secret/
├── configmap.yaml     # Non-sensitive settings
├── secret.yaml        # Sensitive settings
├── pod-env.yaml       # Pod that consumes both
└── README.md
```

---

## ✅ Prerequisites

- A running Kubernetes cluster (Minikube, Kind, Docker Desktop, or cloud)
- `kubectl` installed and configured

Check your setup:

```bash
kubectl cluster-info
kubectl get nodes
```

---

## 🚀 Step-by-Step Walkthrough

### Step 1 – Create the ConfigMap

```bash
kubectl apply -f configmap.yaml
kubectl get configmap pizza-config
```

### Step 2 – Create the Secret

```bash
kubectl apply -f secret.yaml
kubectl get secret pizza-secret
```

### Step 3 – Create the Pod

```bash
kubectl apply -f pod-env.yaml
kubectl get pods
```

Wait until the status is `Running`.

> 💡 **Order matters!** The ConfigMap and Secret must exist **before** the Pod starts, otherwise the Pod will fail with `CreateContainerConfigError`.

### Step 4 – Enter the Pod and check the variables

```bash
kubectl exec -it pizza-pod -- sh
```

Inside the container:

```sh
env | grep PIZZA
env | grep DB_PASSWORD
```

Expected output:

```
PIZZA_TYPE=Cheese
PAYMENT_MODE=Cash
DB_PASSWORD=superuser
```

Type `exit` to leave the container.

> ✍️ Note the space: it is `-- sh` (two dashes, space, then the command). Writing `--sh` is a common typo and will fail.

---

## 🔍 Understanding Each File

### 1️⃣ `configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pizza-config
data:
  PIZZA_TYPE: Cheese
  PAYMENT_MODE: Cash
```

| Field | Meaning |
|---|---|
| `apiVersion: v1` | ConfigMap is part of the core API |
| `kind: ConfigMap` | The type of object |
| `metadata.name` | Name used to reference it later |
| `data` | Key-value pairs (plain text) |

---

### 2️⃣ `secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pizza-secret
type: Opaque
data:
  DB_PASSWORD: c3VwZXJ1c2Vy
```

| Field | Meaning |
|---|---|
| `kind: Secret` | ⚠️ must be lowercase `kind` (not `Kind`) |
| `type: Opaque` | Generic user-defined secret (default type) |
| `data` | Values **must be base64 encoded** |

**How to create the base64 value:**

```bash
echo -n "superuser" | base64
# c3VwZXJ1c2Vy
```

> Always use `-n`. Without it, `echo` adds a newline and your password becomes `superuser\n`.

**How to decode it:**

```bash
echo "c3VwZXJ1c2Vy" | base64 --decode
# superuser
```

**Easier alternative – `stringData`** (Kubernetes encodes it for you):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pizza-secret
type: Opaque
stringData:
  DB_PASSWORD: superuser
```

---

### 3️⃣ `pod-env.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pizza-pod
spec:
  containers:
    - name: pizza-container
      image: busybox
      command: ["sh", "-c", "sleep 3000"]
      envFrom:
        - configMapRef:
            name: pizza-config
        - secretRef:
            name: pizza-secret
```

| Field | Meaning |
|---|---|
| `command: ["sh","-c","sleep 3000"]` | Keeps the container alive for 50 minutes so we can inspect it |
| `envFrom` | Loads **all** keys from a source as environment variables |
| `configMapRef` | Take every key from ConfigMap `pizza-config` |
| `secretRef` | Take every key from Secret `pizza-secret` |

**What happens at Pod start:**

```
ConfigMap (PIZZA_TYPE, PAYMENT_MODE) ─┐
                                      ├──► Environment variables inside container
Secret    (DB_PASSWORD)             ──┘
```

---

## 🔎 Verify Everything

```bash
# View ConfigMap
kubectl get configmap pizza-config -o yaml
kubectl describe configmap pizza-config

# View Secret (values are hidden in describe)
kubectl get secret pizza-secret -o yaml
kubectl describe secret pizza-secret

# Decode a secret value
kubectl get secret pizza-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode

# Run a single command in the Pod without opening a shell
kubectl exec pizza-pod -- env | grep -E "PIZZA|DB_PASSWORD"

# Debug a Pod that won't start
kubectl describe pod pizza-pod
```

---

## 🧰 Other Ways to Use ConfigMap and Secret

### A) Create them with `kubectl` (no YAML)

```bash
kubectl create configmap pizza-config \
  --from-literal=PIZZA_TYPE=Cheese \
  --from-literal=PAYMENT_MODE=Cash

kubectl create secret generic pizza-secret \
  --from-literal=DB_PASSWORD=superuser
```

You can also create them from files:

```bash
kubectl create configmap app-config --from-file=app.properties
```

### B) Pick individual keys with `env` (instead of `envFrom`)

```yaml
env:
  - name: MY_PIZZA
    valueFrom:
      configMapKeyRef:
        name: pizza-config
        key: PIZZA_TYPE
  - name: MY_PASSWORD
    valueFrom:
      secretKeyRef:
        name: pizza-secret
        key: DB_PASSWORD
```

Use this when you want to **rename** variables or load **only some** keys.

### C) Mount as files (volumes)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pizza-pod-volume
spec:
  containers:
    - name: pizza-container
      image: busybox
      command: ["sh", "-c", "sleep 3000"]
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
        - name: secret-vol
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: pizza-config
    - name: secret-vol
      secret:
        secretName: pizza-secret
```

Check inside the Pod:

```bash
kubectl exec -it pizza-pod-volume -- sh
ls /etc/config          # PAYMENT_MODE  PIZZA_TYPE
cat /etc/config/PIZZA_TYPE
cat /etc/secret/DB_PASSWORD
```

Each key becomes a **file**, and the file content is the **value**.

### Env vs Volume – which one?

| | Environment variables | Volume mount |
|---|---|---|
| Simple to use | ✅ | ➖ |
| Auto-updates when ConfigMap/Secret changes | ❌ (needs Pod restart) | ✅ (updates after a short delay) |
| Good for config files (nginx.conf, etc.) | ❌ | ✅ |
| Env vars may leak in logs / crash dumps | ⚠️ yes | Less likely |

---

## 🐞 Common Mistakes

| Problem | Cause | Fix |
|---|---|---|
| `error: unknown field "Kind"` or kind missing | Wrote `Kind` instead of `kind` | YAML keys are case-sensitive, use lowercase `kind` |
| `CreateContainerConfigError` | ConfigMap/Secret doesn't exist or has wrong name | Apply them first, check the names |
| `exec ... --sh` fails | Missing space after `--` | Use `kubectl exec -it pizza-pod -- sh` |
| Wrong password inside the app | Encoded with `echo` without `-n` | Use `echo -n "value" \| base64` |
| Changed ConfigMap but Pod still shows old value | Env vars are read only at start | Delete and recreate the Pod (or use a Deployment rollout) |
| Secret `data` rejected | Value is not valid base64 | Use `stringData` or re-encode |

---

## 🔐 Security Notes

1. **Base64 ≠ encryption.** Anyone with access to `kubectl get secret -o yaml` can decode values.
2. **Never commit real secrets to GitHub.** The example here is for learning only. Add real secret files to `.gitignore`.
3. **Use RBAC** to restrict who can read Secrets.
4. **Enable encryption at rest** for etcd in production.
5. Consider tools like **Sealed Secrets**, **External Secrets Operator**, **HashiCorp Vault** or cloud secret managers.
6. Prefer **volume mounts** over environment variables for highly sensitive values.
7. Use `immutable: true` on ConfigMaps/Secrets that should never change:

```yaml
immutable: true
```

---

## 📝 Cheat Sheet

```bash
# Create
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f pod-env.yaml

# List
kubectl get cm
kubectl get secrets
kubectl get pods

# Inspect
kubectl get cm pizza-config -o yaml
kubectl get secret pizza-secret -o yaml
kubectl describe pod pizza-pod

# Use
kubectl exec -it pizza-pod -- sh
kubectl exec pizza-pod -- env

# Encode / decode
echo -n "superuser" | base64
echo "c3VwZXJ1c2Vy" | base64 --decode

# Delete
kubectl delete -f pod-env.yaml
kubectl delete -f secret.yaml
kubectl delete -f configmap.yaml
```

---

## 🧹 Cleanup

```bash
kubectl delete -f pod-env.yaml
kubectl delete -f secret.yaml
kubectl delete -f configmap.yaml
```

Or in one command:

```bash
kubectl delete pod pizza-pod && kubectl delete cm pizza-config && kubectl delete secret pizza-secret
```

---

## 🎯 Practice Challenges

1. Change `PIZZA_TYPE` to `Pepperoni`, re-create the Pod, and verify the new value.
2. Add a new key `SIZE: Large` to the ConfigMap and confirm it appears in the Pod.
3. Rewrite `secret.yaml` using `stringData`.
4. Load only `PIZZA_TYPE` with `configMapKeyRef` and rename it to `MY_PIZZA`.
5. Mount the ConfigMap as a volume and `cat` each file.
6. Update the ConfigMap while a volume-mounted Pod is running and watch the file change.

---

## 📖 Further Reading

- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Good practices for Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

---

⭐ If this helped you, give the repo a star and happy learning! 🚀