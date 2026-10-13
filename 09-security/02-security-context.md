# Security Context

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../labs/08-rbac/)

[← RBAC](RBAC/README.md) | [README](README.md) | [Pod Security →](pod-security.md)

---

## Idea

RBAC controls who can use the **API**. A **security context** controls what a **container is allowed to do on the node**: which user it runs as, which Linux privileges it has, whether it can write to its filesystem. It is how you limit the damage if a container is compromised.

```
Attacker gets a shell in your container
   ├─ running as root with all capabilities, writable filesystem  → can do a lot, maybe escape
   └─ non-root, no capabilities, read-only filesystem             → can do very little
```

## Where it goes

| Level | Field | Applies to |
|---|---|---|
| **Pod** | `spec.securityContext` | All containers (and volumes) in the Pod |
| **Container** | `spec.containers[].securityContext` | One container. **Overrides** the Pod value |

| Setting | Pod level | Container level |
|---|---|---|
| `runAsUser`, `runAsGroup`, `runAsNonRoot` | Yes | Yes |
| `seccompProfile`, `seLinuxOptions` | Yes | Yes |
| `fsGroup`, `fsGroupChangePolicy`, `supplementalGroups`, `sysctls` | **Pod only** | No |
| `allowPrivilegeEscalation`, `privileged`, `readOnlyRootFilesystem`, `capabilities`, `procMount` | No | **Container only** |

## The settings that matter

### 1. Run as a non-root user

```yaml
securityContext:
  runAsNonRoot: true       # refuse to start if the image would run as root
  runAsUser: 10001
  runAsGroup: 10001
```

Root inside a container is **root-like** on the node if something goes wrong. Non-root is the single most valuable setting.

- `runAsNonRoot: true` makes the kubelet **check** the image user. A root image fails with `container has runAsNonRoot and image will run as root`.
- If the image's `USER` is a name (not a number), Kubernetes cannot verify it. Set `runAsUser` or use a numeric `USER` in the Dockerfile.

### 2. Block privilege escalation

```yaml
securityContext:
  allowPrivilegeEscalation: false      # no gaining more privileges (setuid binaries, sudo)
```

Sets the Linux `no_new_privs` flag on the process.

### 3. Drop Linux capabilities

Root's powers are split into **capabilities**. Containers get a default set (such as `CHOWN`, `SETUID`, `SETGID`, `NET_RAW`, `KILL`, `NET_BIND_SERVICE`). Drop them all, then add back only what you need:

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]          # only if you must bind a port below 1024
```

Most applications need no capabilities at all. Easiest of all: listen on a port above 1024 and add nothing.

### 4. Read-only root filesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp                    # writable scratch space where the app needs it
volumes:
  - name: tmp
    emptyDir: {}
```

An attacker cannot drop tools or change binaries. Give the app `emptyDir` volumes for the few places it must write (`/tmp`, caches, pid files).

### 5. Seccomp: filter system calls

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault               # the runtime's default filter (blocks risky syscalls)
```

Without a seccomp profile, a container may run **Unconfined** (every syscall allowed) unless the cluster enables a default.

### 6. Volume ownership

```yaml
spec:
  securityContext:
    fsGroup: 10001                     # mounted volumes become group-owned by this GID
```

See [Volumes](../06-storage/volumes.md).

### 7. Never do these without a strong reason

| Setting | Why it is dangerous |
|---|---|
| `privileged: true` | Nearly all host devices and capabilities. Close to root on the node |
| `hostNetwork`, `hostPID`, `hostIPC: true` | Shares the node's network, process list or IPC namespace (Pod `spec` fields) |
| `hostPath` volumes | Direct access to node files |
| `capabilities.add: ["SYS_ADMIN"]` (or `ALL`) | Almost as powerful as privileged |
| `runAsUser: 0` | Root |

### 8. More options (know they exist)

| Option | Purpose |
|---|---|
| `seLinuxOptions`, AppArmor | Mandatory access control profiles |
| `sysctls` | Kernel parameters (a safe subset is allowed) |
| `procMount: Unmasked` | Exposes more of `/proc`. Avoid |
| `hostUsers: false` | **User namespaces**: root in the container maps to an unprivileged user on the node (newer clusters) |

## A hardened Pod template

This passes the **Restricted** Pod Security profile:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secctx-hardened
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

### In your Dockerfile (Node.js)

```dockerfile
FROM node:22-slim
WORKDIR /app
COPY --chown=1000:1000 . .
USER 1000                       # numeric non-root user (the "node" user)
EXPOSE 3000                     # port above 1024
CMD ["node", "dist/main.js"]
```

Make the app write only to `/tmp` or another mounted `emptyDir`.

## Lab

```bash
cd labs/08-rbac/manifests
kubectl apply -f secctx-demo.yaml                        # secctx-default and secctx-hardened
kubectl wait --for=condition=Ready pod/secctx-default pod/secctx-hardened --timeout=60s
```

**1. Who am I?**

```bash
kubectl exec secctx-default -- id                        # uid=0(root)
kubectl exec secctx-hardened -- id                       # uid=10001 gid=10001
```

**2. Capabilities, privilege escalation and seccomp, as the kernel sees them**

```bash
for p in secctx-default secctx-hardened; do
  echo "--- $p"
  kubectl exec $p -- grep -E 'CapEff|NoNewPrivs|Seccomp:' /proc/1/status
done
```

Compare: the default Pod has a non-zero `CapEff`, `NoNewPrivs: 0`, and `Seccomp: 0` (unfiltered). The hardened Pod shows `CapEff` all zeros, `NoNewPrivs: 1` and `Seccomp: 2` (filtered).

**3. Read-only root filesystem**

```bash
kubectl exec secctx-default -- sh -c 'touch /file && echo "default: wrote /file"'
kubectl exec secctx-hardened -- sh -c 'touch /file'                       # Read-only file system
kubectl exec secctx-hardened -- sh -c 'touch /tmp/file && echo "hardened: wrote /tmp/file"'
```

**4. `runAsNonRoot` catches a root image**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-check
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: web
      image: nginx:1.27            # this image runs as root
EOF
kubectl get pod nonroot-check                                              # CreateContainerConfigError
kubectl describe pod nonroot-check | grep -A3 -i "runAsNonRoot"            # image will run as root
```

**5. A privileged Pod sees the node's devices** (a demonstration of the risk)

```bash
kubectl apply -f pod-privileged.yaml
kubectl exec priv-pod -- sh -c 'ls /dev | wc -l'                           # many host devices
kubectl exec secctx-hardened -- sh -c 'ls /dev | wc -l'                    # only a handful
kubectl delete -f pod-privileged.yaml
```

**Cleanup**

```bash
kubectl delete -f secctx-demo.yaml
kubectl delete pod nonroot-check --ignore-not-found
```

## Gotchas

- `runAsNonRoot: true` with an image that has a **named** non-root user fails verification. Use a numeric `USER`.
- A **read-only root filesystem** breaks apps that write to unexpected places. Run with it on, watch for `Read-only file system` errors and add `emptyDir` mounts.
- **Dropping all capabilities** can break binding to low ports and some tools. Add back only what is needed.
- Container-level settings **override** Pod-level ones for that container.
- A volume that is root-owned may not be writable by a non-root user: use `fsGroup`.
- A **security context does not replace RBAC or network policy**. Each controls a different layer.
- The kubelet enforces these settings, but **admission** (see [Pod Security](pod-security.md)) is what makes sure nobody **skips** them.

## Check yourself

1. What does a security context control that RBAC does not?
2. What does `runAsNonRoot: true` do when the image runs as root?
3. Why drop all capabilities?
4. What does `readOnlyRootFilesystem: true` protect against, and what must you add for the app?
5. Which settings exist only at Pod level?
6. Why is `privileged: true` dangerous?

<details>
<summary>Answers</summary>

1. What a container may do on the node: user, Linux privileges, filesystem and syscalls (RBAC controls API access)
2. The container fails to start (`CreateContainerConfigError`: image will run as root)
3. Most apps need none, and each capability gives an attacker more power. Add back only what is required
4. Tampering with binaries and dropping tools. Add `emptyDir` volumes where the app must write (such as `/tmp`)
5. `fsGroup`, `fsGroupChangePolicy`, `supplementalGroups`, `sysctls`
6. It gives the container almost all host devices and capabilities, close to root on the node

</details>

---

[← RBAC](RBAC/README.md) | [README](README.md) | [Pod Security →](pod-security.md)
