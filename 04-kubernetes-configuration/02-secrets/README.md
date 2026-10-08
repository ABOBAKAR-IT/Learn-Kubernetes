# Secrets

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/04-configmap-secret/manifests/`](../labs/04-configmap-secret/manifests/)

[← ConfigMaps](configmaps.md) | [README](README.md) | [Environment Variables →](environment-variables.md)

---

## Idea

A **Secret** stores **sensitive data**: passwords, API tokens, TLS keys, registry credentials. It works like a ConfigMap (key/value pairs, used as env vars or files), with extra handling:

| Compared with a ConfigMap | Secret |
|---|---|
| Values stored as | **base64** text in `data` |
| `kubectl describe` | Shows key names and sizes, **not** values |
| Mounted as a volume | Kept in **memory (tmpfs)** on the node, not on disk |
| Access control | You can give RBAC access to ConfigMaps but not Secrets |
| Size limit | 1 MiB |

## The most important fact

> **base64 is encoding, not encryption.** Anyone who can read a Secret can decode it in one command.

```bash
echo 'UzNjcjN0IQ==' | base64 -d        # S3cr3t!
```

Out of the box:

- Secrets sit **unencrypted in etcd**.
- Anyone with `get` access to Secrets in a namespace can read them.
- Any Pod in the namespace can mount them.

So a Secret is only as safe as your **RBAC, etcd encryption and Git hygiene**.

## Creating Secrets

### Imperative

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='S3cr3t!'

kubectl create secret generic app-keys --from-file=./api.key          # key = file name
```

### YAML: `stringData` (plain text, easiest) or `data` (base64)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                 # write-only convenience, Kubernetes stores it in data as base64
  username: admin
  password: change-me-demo-only
# data:                     # alternatively, values you encoded yourself
#   username: YWRtaW4=
```

Encode and decode by hand (note `-n` so no trailing newline is included):

```bash
echo -n 'admin' | base64                                         # YWRtaW4=
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

> Never commit real values. Files with `stringData` are for demos only.

## Secret types

| Type | Purpose | Create with |
|---|---|---|
| `Opaque` (default) | Any key/value data | `kubectl create secret generic` |
| `kubernetes.io/tls` | TLS certificate and key (`tls.crt`, `tls.key`) | `kubectl create secret tls site-tls --cert=tls.crt --key=tls.key` |
| `kubernetes.io/dockerconfigjson` | Private image registry login | `kubectl create secret docker-registry regcred --docker-server=... --docker-username=... --docker-password=...` |
| `kubernetes.io/basic-auth` | Username and password | YAML with `type:` set |
| `kubernetes.io/ssh-auth` | SSH private key | YAML with `type:` set |
| `kubernetes.io/service-account-token` | ServiceAccount token | Managed by Kubernetes |

The type enforces required keys (a TLS Secret must have `tls.crt` and `tls.key`).

## Using a Secret in a Pod

### As an environment variable

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

### All keys as environment variables

```yaml
envFrom:
  - secretRef:
      name: db-secret
```

### As files (preferred)

```yaml
containers:
  - name: app
    volumeMounts:
      - name: secret-vol
        mountPath: /etc/secret
        readOnly: true
volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
      defaultMode: 0400              # owner read-only
```

Each key becomes a file (`/etc/secret/username`, `/etc/secret/password`) holding the decoded value.

### Private registry login for image pulls

```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: registry.example.com/team/app:1.0
```

(You can also attach `imagePullSecrets` to a ServiceAccount so every Pod using it gets the login.)

## Env vars or files?

| | Environment variable | Mounted file |
|---|---|---|
| Updates when the Secret changes | No (needs restart) | Yes, after about a minute |
| Leak risk | Higher: appears in crash dumps, `/proc`, child processes, debug output | Lower: file permissions apply |
| Ease for the app | `process.env.X` | Read a file |
| Recommended for | Low-risk values, apps that only read env | Passwords, keys, certificates |

## Making Secrets actually safe

| Practice | How |
|---|---|
| **Encrypt at rest** | Enable etcd encryption. On EKS use envelope encryption with a KMS key |
| **Least-privilege RBAC** | Few subjects can `get`, `list` or `watch` Secrets. Remember `list` returns the data too |
| **Keep them out of Git** | Use **Sealed Secrets** or **SOPS** to store encrypted files, or an external store |
| **Use an external secret manager** | AWS Secrets Manager or SSM Parameter Store through the **External Secrets Operator** or the **Secrets Store CSI driver** |
| **Prefer identity over keys** | On AWS give Pods an IAM role (IRSA) instead of storing access keys |
| **Rotate and audit** | Short-lived credentials, audit logs on Secret access |
| **One Secret per purpose** | Smaller blast radius, simpler RBAC |
| **Do not log them** | Never print env vars or request bodies containing secrets |

## Lab

Manifests: `db-secret.yaml`, `secret-pod.yaml`.

```bash
cd labs/04-configmap-secret/manifests

# 1. Create and look at it
kubectl apply -f db-secret.yaml
kubectl get secret db-secret
kubectl describe secret db-secret                     # key names and sizes, no values
kubectl get secret db-secret -o yaml                  # values are only base64
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d; echo

# 2. A Pod using the Secret as env var and as files
kubectl apply -f secret-pod.yaml
kubectl logs secret-demo
kubectl exec secret-demo -- ls -l /etc/secret          # files with restricted mode
kubectl exec secret-demo -- cat /etc/secret/password; echo
kubectl exec secret-demo -- mount | grep secret        # tmpfs: kept in memory

# 3. See why env vars leak more easily
kubectl exec secret-demo -- env | grep DB_             # visible to anyone who can exec

# 4. Who may read Secrets here?
kubectl auth can-i get secrets
kubectl auth can-i get secrets --as=system:serviceaccount:default:default

# 5. Change the Secret
kubectl patch secret db-secret --type merge -p '{"stringData":{"password":"new-demo-password"}}'
kubectl exec secret-demo -- printenv DB_PASSWORD       # env unchanged until restart
sleep 70
kubectl exec secret-demo -- cat /etc/secret/password; echo   # file updated

# 6. Create a TLS and a registry Secret (practice commands)
openssl req -x509 -newkey rsa:2048 -nodes -keyout tls.key -out tls.crt -subj "/CN=demo.local" -days 1
kubectl create secret tls demo-tls --cert=tls.crt --key=tls.key
kubectl get secret demo-tls -o jsonpath='{.type}{"\n"}'
rm tls.key tls.crt

# Cleanup
kubectl delete pod secret-demo
kubectl delete secret db-secret demo-tls
```

## Gotchas

- **base64 is not security.** Treat a Secret like plain text for anyone with access.
- `echo 'admin' | base64` includes a trailing newline and breaks logins. Use `echo -n`.
- `kubectl describe` hides values but `kubectl get secret -o yaml` shows them.
- A Secret change does **not** restart Pods. Env vars stay stale until a rollout.
- Secrets are namespaced, so a Pod cannot use a Secret from another namespace.
- Anyone who can create a Pod in the namespace can mount any Secret there. RBAC on **Pods** matters too.

## Check yourself

1. Is a Secret encrypted? What does `base64` do?
2. What is the difference between `data` and `stringData`?
3. Why are mounted Secret files safer than env vars?
4. Name two ways to keep Secrets out of Git.
5. Which Secret type is used for pulling private images?
6. How do you read a Secret value from the command line?

<details>
<summary>Answers</summary>

1. Not by default. base64 only encodes the value, and it is reversible with one command. Encryption at rest must be enabled separately
2. `data` takes base64 values. `stringData` takes plain text and is converted to `data` on write
3. Files have permissions, live in memory, update on change and leak less through crash dumps and process inspection
4. Sealed Secrets, SOPS, or an external manager through External Secrets Operator / Secrets Store CSI
5. `kubernetes.io/dockerconfigjson`
6. `kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d`

</details>

---

[← ConfigMaps](configmaps.md) | [README](README.md) | [Environment Variables →](environment-variables.md)