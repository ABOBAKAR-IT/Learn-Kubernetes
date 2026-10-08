# Environment Variables

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/04-configmap-secret/manifests/`](../labs/04-configmap-secret/manifests/)

[← Secrets](secrets.md) | [README](README.md)

---

## Idea

Environment variables are the most common way an application receives configuration. Kubernetes can fill them from **five different sources**, and this note shows how they combine.

Your code just reads them:

```ts
// NestJS / Node.js: nothing Kubernetes-specific
const port = parseInt(process.env.PORT ?? '3000', 10);
const logLevel = process.env.LOG_LEVEL ?? 'info';
```

## The five sources

| Source | YAML | Use for |
|---|---|---|
| **Literal value** | `value: "hello"` | Fixed, non-secret settings |
| **ConfigMap key** | `valueFrom.configMapKeyRef` | One setting from a ConfigMap |
| **Secret key** | `valueFrom.secretKeyRef` | One sensitive value |
| **Pod metadata** | `valueFrom.fieldRef` | Pod name, IP, node, namespace, labels |
| **Container resources** | `valueFrom.resourceFieldRef` | CPU and memory requests or limits |
| **Whole ConfigMap or Secret** | `envFrom` | Every key at once |

```yaml
containers:
  - name: app
    image: busybox:1.36
    envFrom:                                 # bulk: all keys of a ConfigMap and a Secret
      - configMapRef:
          name: env-config
      - secretRef:
          name: env-secret
        prefix: SECRET_                      # SECRET_API_TOKEN
    env:                                     # individual variables
      - name: GREETING
        value: "hello"
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
      - name: MEMORY_LIMIT_MI
        valueFrom:
          resourceFieldRef:
            containerName: app
            resource: limits.memory
            divisor: 1Mi
      - name: MESSAGE
        value: "$(GREETING) from $(POD_NAME)"   # expands earlier variables
```

## The Downward API

The **Downward API** lets a container learn about **its own Pod** without calling the API server.

| `fieldPath` | Gives |
|---|---|
| `metadata.name` | Pod name |
| `metadata.namespace` | Namespace |
| `metadata.labels['app']` | One label |
| `metadata.annotations['key']` | One annotation |
| `spec.nodeName` | Node it runs on |
| `spec.serviceAccountName` | ServiceAccount |
| `status.podIP` | Pod IP |
| `status.hostIP` | Node IP |

Use it for logging context (which Pod wrote this line), service registration, and per-Pod identifiers (for example in a StatefulSet, `POD_NAME` is the stable `db-0`). Labels and annotations can also be exposed as **files** with a `downwardAPI` volume, and those files update live while env vars do not.

## Rules when sources overlap

1. **`env` beats `envFrom`.** An explicit entry overrides a key from a ConfigMap or Secret.
2. Between several `envFrom` sources, **the last one wins** when a key repeats.
3. `prefix` on an `envFrom` item avoids clashes (`CFG_LOG_LEVEL`).
4. A key that is not a valid variable name is **skipped** (the Pod still starts and an event is recorded).
5. A missing ConfigMap or Secret blocks the Pod (`CreateContainerConfigError`) unless you mark it `optional: true`.

## Expanding variables: `$(VAR)`

```yaml
env:
  - name: HOST
    value: "db.internal"
  - name: DB_URL
    value: "postgres://$(HOST):5432/app"     # HOST is defined above, so this works
args: ["--db=$(DB_URL)"]                      # also works in command and args
```

- Only variables defined **earlier in the same `env` list** can be used.
- Unknown names stay literal text. Escape with `$$(VAR)` to print `$(VAR)`.
- `$VAR` (no parentheses) is **not** expanded by Kubernetes. It is left for the shell.

## Variables Kubernetes adds for Services

For every Service that exists **before the Pod starts**, Kubernetes injects variables like `WEB_SERVICE_HOST` and `WEB_SERVICE_PORT`.

- They are a legacy mechanism. **Use DNS** (`http://web`) instead.
- Order matters: Services created after the Pod are missing, which causes confusing bugs.
- With many Services they clutter the environment. Disable with `enableServiceLinks: false` in the Pod spec.

## Env vs files: choosing

| Need | Choose |
|---|---|
| The app reads `process.env` | Environment variable |
| Settings must update without restart | Mounted file (and a way to reload) |
| Whole config file (nginx.conf, application.yaml) | Mounted file from a ConfigMap |
| Passwords, keys, certificates | Secret as a **file** (see [Secrets](secrets.md)) |
| Pod identity (name, IP, node) | Downward API |
| Value differs per environment | ConfigMap per environment, same image |

## Limits and gotchas

- Environment variables are **fixed at container start**. Changing the ConfigMap or Secret needs a restart (`kubectl rollout restart`).
- Values are **strings**. Quote numbers and booleans in YAML (`"8080"`, `"true"`), or the API rejects the manifest.
- Variable **names are case sensitive** and cannot start with a digit.
- Anyone who can `kubectl exec` can run `env`. Secrets in env vars are visible to them.
- A very large number of variables or very long values slows container start and can hit OS limits.

## Lab

Manifest: `env-demo.yaml` (a ConfigMap, a Secret and a Pod, all self-contained).

```bash
cd labs/04-configmap-secret/manifests
kubectl apply -f env-demo.yaml
kubectl logs env-demo | sort
```

Check each line of the output:

| Variable | Comes from |
|---|---|
| `APP_ENV=dev`, `REGION=...` | `envFrom` the ConfigMap |
| `LOG_LEVEL=override` | The explicit `env` entry beats `envFrom` (the ConfigMap says `warn`) |
| `SECRET_API_TOKEN=...` | `envFrom` the Secret with `prefix: SECRET_` |
| `POD_NAME`, `POD_IP`, `NODE_NAME` | Downward API (`fieldRef`) |
| `MEMORY_LIMIT_MI=128` | `resourceFieldRef` with divisor `1Mi` |
| `MESSAGE=hello from env-demo` | `$(VAR)` expansion |

```bash
# Prove the rules
kubectl exec env-demo -- printenv LOG_LEVEL             # override
kubectl exec env-demo -- env | grep -c '^WEB_'          # 0 because enableServiceLinks is false

# Env vars do not follow ConfigMap changes
sed 's/APP_ENV: dev/APP_ENV: prod/' env-demo.yaml | kubectl apply -f -   # updates the ConfigMap only
kubectl exec env-demo -- printenv APP_ENV               # still "dev": env is fixed at container start
kubectl delete pod env-demo
sed 's/APP_ENV: dev/APP_ENV: prod/' env-demo.yaml | kubectl apply -f -   # recreates the Pod
kubectl wait --for=condition=Ready pod/env-demo --timeout=60s
kubectl exec env-demo -- printenv APP_ENV               # "prod": a new container reads the new value
kubectl delete -f env-demo.yaml
```

Break it on purpose:

```bash
# 1. a boolean without quotes is rejected: change `value: hello` to `value: true`, then apply
# 2. reference a key that does not exist (key: NOPE) and watch CreateContainerConfigError
```

## Check yourself

1. Name the five sources of an environment variable.
2. `env` and `envFrom` both define `LOG_LEVEL`. Which wins?
3. What does `prefix` do in `envFrom`?
4. Is `$HOME` in a container `args` list expanded by Kubernetes?
5. How can a container learn its own Pod name?
6. Why can an environment variable not follow a ConfigMap change?

<details>
<summary>Answers</summary>

1. Literal value, ConfigMap key, Secret key, Pod/resource fields (Downward API), and bulk `envFrom` from a whole ConfigMap or Secret
2. The explicit `env` entry
3. Adds a prefix to every key from that source, preventing name clashes
4. No. Kubernetes expands only `$(VAR)`. `$HOME` is left to the shell
5. `valueFrom.fieldRef.fieldPath: metadata.name`
6. The environment is built once when the container starts, so a restart is needed

</details>

---

[← Secrets](secrets.md) | [README](README.md)