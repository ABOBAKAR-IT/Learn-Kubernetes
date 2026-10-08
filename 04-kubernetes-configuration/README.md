# Module 04: Kubernetes Configuration

> Separate configuration from code with ConfigMaps, Secrets and environment variables.

**Notes ready:** 4/4 topics

## Topic directory

| Topic | File | Question it answers | Status |
|---|---|---|---|
| **ConfigMaps** | [`configmaps.md`](configmaps.md) | How do I keep non-secret settings outside the image? | 📘 |
| **Secrets** | [`secrets.md`](secrets.md) | How do I handle passwords and keys, and how safe is it really? | 📘 |
| **Environment Variables** | [`environment-variables.md`](environment-variables.md) | How does config reach my application, and which source wins? | 📘 |
| **YAML Manifests** | [`yaml-manifests.md`](yaml-manifests.md) | How do I write, validate and organize manifests without errors? | 📘 |

YAML files for every lab: [`labs/04-configmap-secret/manifests/`](../labs/04-configmap-secret/manifests/).

## Recommended study order

| # | Topic | Why now |
|---|---|---|
| 1 | [YAML Manifests](yaml-manifests.md) | The skill you use in every other note |
| 2 | [ConfigMaps](configmaps.md) | The basic building block for settings |
| 3 | [Secrets](secrets.md) | The same idea with a security lens |
| 4 | [Environment Variables](environment-variables.md) | Ties it all together: how config reaches the container |

## The big idea: one image, many environments

```
                 same container image: my-api:1.4.2
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
   dev namespace          staging namespace        prod namespace
   ConfigMap (dev)        ConfigMap (staging)      ConfigMap (prod)
   Secret (dev)           Secret (staging)         Secret (prod)
```

The image never changes. Kubernetes injects the configuration at runtime:

```
ConfigMap ─┐                      ┌─► environment variables
           ├──► Pod template ─────┤
Secret ────┘                      └─► files in a volume
```

## Where should this value live?

| The value is... | Put it in |
|---|---|
| A non-sensitive setting (log level, feature flag, URL) | **ConfigMap** |
| A whole config file (nginx.conf, application.yaml) | **ConfigMap**, mounted as a file |
| A password, token, key or certificate | **Secret** (ideally an external manager) |
| Information about the Pod itself (name, IP, node) | **Downward API** (`fieldRef`) |
| Needs to change without restarting | A **mounted file** plus a reload mechanism |
| Fixed at build time and never changes | The image itself |

## ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| For | Non-sensitive config | Sensitive data |
| Stored as | Plain text | base64 (not encryption) |
| Mounted files | On the node's disk | In memory (tmpfs) |
| `kubectl describe` | Shows values | Hides values |
| Size limit | 1 MiB | 1 MiB |
| Needs extra protection | No | **Yes**: etcd encryption, RBAC, no Git, external manager |

## How config reaches the container

| Method | Updates without restart | Best for |
|---|---|---|
| Environment variable | No | Simple settings the app reads from `process.env` |
| Mounted file | Yes (about a minute, not with `subPath`) | Config files, certificates, passwords |
| Command arguments | No | Flags such as `--log-level=$(LOG_LEVEL)` |

## Related labs

- [labs/04-configmap-secret](../labs/04-configmap-secret/): a web server configured by a ConfigMap and a Secret, with live changes, reloads and rotation

## Module quiz

1. Why keep configuration outside the container image?
2. Name two differences between a ConfigMap and a Secret.
3. Is base64 a form of encryption?
4. You edit a ConfigMap. Which consumers see the change without a restart?
5. Which kind of mount never receives ConfigMap updates?
6. `env` and `envFrom` define the same variable. Which wins?
7. How can a container learn its own Pod name?
8. What error shows when a manifest has `value: true` for an env var?
9. Which command validates a manifest against the real API without saving it?
10. How do you trigger a rolling restart after a config change?
11. Which Secret type holds a TLS certificate and key?
12. Why are mounted Secret files safer than environment variables?
13. What does `immutable: true` do?
14. Which command shows what would change before you apply a manifest?

<details>
<summary>Answers</summary>

1. One image runs in every environment, and config changes need no rebuild
2. Secrets are base64-encoded, hidden in `describe`, kept in memory when mounted, and need extra protection. ConfigMaps are plain text for non-sensitive data
3. No. It is reversible encoding with no key
4. Volume-mounted files (not `subPath`), after about a minute. Environment variables and the running app's in-memory config do not
5. `subPath` mounts
6. The explicit `env` entry
7. Downward API: `valueFrom.fieldRef.fieldPath: metadata.name`
8. `cannot unmarshal bool into ... of type string`. Fix with `value: "true"`
9. `kubectl apply -f file.yaml --dry-run=server`
10. `kubectl rollout restart deployment/<name>`
11. `kubernetes.io/tls`
12. They have file permissions, live in memory, update on change and leak less through crash dumps, `env` output and child processes
13. Makes the ConfigMap's data unchangeable; create a new versioned ConfigMap instead
14. `kubectl diff -f file.yaml`

</details>

## Self-test before moving on

1. Deploy `app-stack.yaml` from memory in a new namespace and change its config without looking at the notes.
2. Explain what a stolen kubeconfig with `get secrets` permission can do, and three ways to reduce that risk.
3. List the five sources of an environment variable and say which ones update on restart only.
4. Break `broken-deployment.yaml` again in a new way and fix it using only `--dry-run=server` and `kubectl explain`.

## How to study this module

1. Read each note in order.
2. Type the YAML yourself, do not copy-paste.
3. Run the lab, then break it on purpose.
4. When you have studied and practiced a note, change its `Status` line to `✅ Studied`.

Next: [Module 05: Networking](../05-kubernetes-networking/) (recommended) or [Module 06: Storage](../06-storage/)