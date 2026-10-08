# ConfigMaps

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/04-configmap-secret/manifests/`](../labs/04-configmap-secret/manifests/)

[← YAML Manifests](yaml-manifests.md) | [README](README.md) | [Secrets →](secrets.md)

---

## Idea

A **ConfigMap** stores **non-sensitive configuration** as key/value pairs, **outside** your container image.

```
Without ConfigMap:  change a setting → rebuild image → push → redeploy
With ConfigMap:     change the ConfigMap → restart (or reload) the Pods
```

This is the "config in the environment" rule of twelve-factor apps: **one image, many environments** (dev, staging, prod), each with its own ConfigMap.

| Fact | Value |
|---|---|
| Namespaced | Yes. A Pod can only use ConfigMaps from its own namespace |
| Size limit | 1 MiB |
| Key names | Letters, digits, `-`, `_`, `.` |
| Not for | Passwords, tokens, keys (use [Secrets](secrets.md)) |

## Creating a ConfigMap

### 1. From literals

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=APP_ENV=dev
```

### 2. From a file (the key is the file name)

```bash
kubectl create configmap nginx-conf --from-file=default.conf
kubectl create configmap nginx-conf --from-file=site.conf=default.conf   # choose the key name
kubectl create configmap all-conf --from-file=./conf-dir/                # every file in a folder
```

### 3. From an env file (`KEY=value` per line)

```bash
kubectl create configmap env-conf --from-env-file=.env
```

### 4. From YAML (best for Git)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"            # simple key
  APP_ENV: "dev"
  app.properties: |            # a whole file as one key
    cache.ttl=60
    feature.newUI=false
```

`data` holds text. `binaryData` (base64) holds binary content.

Inspect it:

```bash
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

## Using a ConfigMap in a Pod: 4 ways

### A. One key as an environment variable

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
```

### B. All keys as environment variables

```yaml
envFrom:
  - configMapRef:
      name: app-config
    prefix: CFG_                 # optional: CFG_LOG_LEVEL, CFG_APP_ENV
```

### C. Mounted as files (every key becomes a file)

```yaml
containers:
  - name: app
    volumeMounts:
      - name: config-vol
        mountPath: /etc/config            # /etc/config/LOG_LEVEL, /etc/config/app.properties
volumes:
  - name: config-vol
    configMap:
      name: app-config
```

Pick only some keys and rename the files:

```yaml
volumes:
  - name: config-vol
    configMap:
      name: app-config
      items:
        - key: app.properties
          path: application.properties    # /etc/config/application.properties
      defaultMode: 0644
```

### D. In command arguments

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef: { name: app-config, key: LOG_LEVEL }
args: ["--log-level=$(LOG_LEVEL)"]       # $(VAR) is expanded by Kubernetes
```

## What happens when the ConfigMap changes?

| How it is used | Picks up the change? |
|---|---|
| Environment variables (`env`, `envFrom`) | **No.** Fixed when the container starts. Restart the Pod |
| Volume mount | **Yes**, after about a minute. The files are updated in place |
| Volume mount with `subPath` | **No**, `subPath` mounts never update |
| Application reads the file once at startup | Updated file, but the app still uses old values until it reloads |

So changing a ConfigMap often needs a restart:

```bash
kubectl rollout restart deployment/web
```

A common pattern is to put a **hash of the config in the Pod template** (Helm: a `checksum/config` annotation, Kustomize: `configMapGenerator`), so that changing the config changes the template and triggers a rolling update automatically.

## Immutable ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
data:
  LOG_LEVEL: info
immutable: true
```

- The data can **never be changed**. Create `app-config-v2` and point the Deployment at it.
- Safer (no accidental edits) and lighter for the cluster (kubelets stop watching it).
- Versioned names plus a rolling update give you easy config rollbacks.

## Lab

Manifests: `app-config.yaml`, `configmap-pod.yaml`, `immutable-config.yaml`.

```bash
cd labs/04-configmap-secret/manifests

# 1. Create and inspect
kubectl apply -f app-config.yaml
kubectl get configmap app-config -o yaml

# 2. A Pod that uses it as an env var and as files
kubectl apply -f configmap-pod.yaml
kubectl logs config-demo --tail=5
kubectl exec config-demo -- ls /etc/config
kubectl exec config-demo -- cat /etc/config/app.properties

# 3. Change the ConfigMap
kubectl patch configmap app-config --type merge \
  -p '{"data":{"LOG_LEVEL":"debug","app.properties":"cache.ttl=120\nfeature.newUI=true\n"}}'

kubectl exec config-demo -- printenv LOG_LEVEL                 # still "info": env is fixed at start
sleep 70
kubectl exec config-demo -- cat /etc/config/app.properties     # updated: files follow the ConfigMap

# 4. Restart to pick up the env change
kubectl delete pod config-demo && kubectl apply -f configmap-pod.yaml
kubectl exec config-demo -- printenv LOG_LEVEL                 # now "debug"

# 5. Immutable
kubectl apply -f immutable-config.yaml
kubectl patch configmap app-config-immutable --type merge -p '{"data":{"LOG_LEVEL":"debug"}}'
# error: field is immutable

# 6. Missing ConfigMap blocks the Pod
kubectl run broken --image=busybox:1.36 --overrides='{"spec":{"containers":[{"name":"broken","image":"busybox:1.36","command":["sleep","3600"],"envFrom":[{"configMapRef":{"name":"does-not-exist"}}]}]}}'
kubectl get pod broken                                         # CreateContainerConfigError

# Cleanup
kubectl delete pod config-demo broken --ignore-not-found
kubectl delete configmap app-config app-config-immutable
```

## Gotchas

- **Env vars from a ConfigMap never update** without a restart. Only volume files follow changes.
- A missing ConfigMap (or key) leaves the Pod in `CreateContainerConfigError`. Use `optional: true` to allow it.
- `subPath` mounts do not receive updates.
- Mounting a ConfigMap over a directory **hides everything already in that directory** in the image.
- Do not put secrets in ConfigMaps. They are plain text and widely readable.
- Values are strings. Quote numbers and booleans: `"60"`, `"true"`.

## Check yourself

1. Why keep configuration outside the image?
2. Name the four ways to consume a ConfigMap.
3. You change a ConfigMap. Which consumers see it without a restart?
4. How do you make a ConfigMap impossible to change?
5. What status does a Pod show when its ConfigMap does not exist?

<details>
<summary>Answers</summary>

1. One image can run in every environment, and config changes do not need a rebuild
2. Single env var (`configMapKeyRef`), all keys as env vars (`envFrom`), mounted files (volume), command arguments via `$(VAR)`
3. Volume-mounted files (not `subPath` and not env vars), after about a minute. The app may still need to reload them
4. Set `immutable: true`
5. `CreateContainerConfigError`

</details>

---

[← YAML Manifests](yaml-manifests.md) | [README](README.md) | [Secrets →](secrets.md)