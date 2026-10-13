# Pod Security

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../labs/08-rbac/)

[← Security Context](security-context.md) | [README](README.md) | [Network Security →](network-security.md)

---

## Idea

A [security context](security-context.md) lets a Pod **ask** to be hardened. Nothing forces anyone to **do** it. **Pod Security** is the admission control that **enforces** a minimum standard for every Pod in a namespace, and rejects Pods that break it.

```
kubectl apply pod.yaml
   ▼
Authentication ─► Authorization (RBAC) ─► ADMISSION  ← Pod Security checks the Pod's spec here
                                             ├─ meets the namespace's profile → created
                                             └─ violates it                   → rejected (or warned)
```

It closes the gap in the [RBAC escalation story](RBAC/best-practices.md): even someone who may create Pods cannot create a privileged one.

## Pod Security Standards: three profiles

| Profile | Intent | Typical for |
|---|---|---|
| **Privileged** | **No restrictions** | System components (CNI, storage drivers), `kube-system` |
| **Baseline** | Blocks **known privilege escalations**, stays easy to adopt | Most ordinary workloads |
| **Restricted** | Follows **current hardening best practice** | Security-sensitive and new workloads |

### What each profile forbids (selected, in short)

| Control | Baseline | Restricted |
|---|---|---|
| `privileged: true` | Forbidden | Forbidden |
| `hostNetwork`, `hostPID`, `hostIPC` | Forbidden | Forbidden |
| `hostPath` volumes | Forbidden | Forbidden |
| `hostPort` | Forbidden | Forbidden |
| Extra capabilities (beyond a short allowed list) | Forbidden | **Must drop `ALL`** (may add back only `NET_BIND_SERVICE`) |
| Unsafe `sysctls`, unmasked `/proc`, some SELinux options | Forbidden | Forbidden |
| Running as **root** | Allowed | **Forbidden**: `runAsNonRoot: true` required |
| `allowPrivilegeEscalation` | Allowed | **Must be `false`** |
| **Seccomp** profile | Not `Unconfined` | **`RuntimeDefault` or `Localhost` required** |
| Volume types | Most allowed | Only `configMap`, `csi`, `downwardAPI`, `emptyDir`, `ephemeral`, `persistentVolumeClaim`, `projected`, `secret` |

The [hardened template](security-context.md#a-hardened-pod-template) satisfies **Restricted**.

## Pod Security Admission (PSA)

PSA is the **built-in admission controller** (stable since Kubernetes 1.25, when PodSecurityPolicy was removed). You turn it on **per namespace with labels**:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### Three modes

| Mode | Effect on a violating Pod |
|---|---|
| **`enforce`** | The Pod is **rejected** |
| **`warn`** | The Pod is created, and the user sees a **warning** |
| **`audit`** | The Pod is created, and a violation is **recorded in the audit log** |

Each mode has its own level, so you can **enforce Baseline while warning and auditing for Restricted**, which is a gentle way to raise the bar.

### Labels

```
pod-security.kubernetes.io/<mode>: <level>           # level: privileged | baseline | restricted
pod-security.kubernetes.io/<mode>-version: <version> # pin the policy version (v1.30) or "latest"
```

Pinning a version keeps behavior stable across cluster upgrades.

### Details worth knowing

- **`enforce` checks Pods.** For Deployments, Jobs and others, `warn` and `audit` still check the **Pod template**, so you see problems when you apply the Deployment, while `enforce` blocks the Pod when the controller creates it (the Deployment shows failed ReplicaSet events).
- Existing Pods are **not** evicted when you add an `enforce` label. They only affect **new** Pods.
- The cluster admin can set **cluster-wide defaults** and **exemptions** (users, namespaces, runtime classes) in the admission configuration.
- On many clusters nothing is enforced **until you label namespaces**, so the default is effectively `privileged`.

## Rolling it out safely

```
1. Check what would break:   kubectl label --dry-run=server --overwrite ns <ns> pod-security.kubernetes.io/enforce=baseline
2. Add warn + audit labels:  people see warnings and the audit log shows the violations; nothing breaks
3. Fix the workloads:        add security contexts, change images, remove hostPath, and so on
4. Add the enforce label:    now violations are rejected
5. Move toward restricted:   repeat for sensitive namespaces
```

Leave `kube-system` and other system namespaces at **privileged** (they need it), and keep application namespaces at Baseline or Restricted.

## Beyond PSA: custom policies

PSA has **fixed** profiles. For rules like "no `:latest` images", "must have resource limits" or "only images from our registry", use a policy engine:

| Tool | Notes |
|---|---|
| **ValidatingAdmissionPolicy** | Built-in, written in CEL expressions (stable since Kubernetes 1.30) |
| **Kyverno** | Policies written as Kubernetes YAML, can also mutate and generate resources |
| **OPA Gatekeeper** | Policies written in Rego |

Use PSA for the baseline of Pod hardening, and a policy engine for organization-specific rules.

## Lab

```bash
cd labs/08-rbac/manifests
kubectl apply -f psa-namespaces.yaml
kubectl get ns -L pod-security.kubernetes.io/enforce psa-baseline psa-restricted
```

`psa-baseline` enforces Baseline and warns on Restricted. `psa-restricted` enforces Restricted.

**1. A privileged Pod is rejected under Baseline**

```bash
kubectl apply -f pod-privileged.yaml -n psa-baseline
# Error ... violates PodSecurity "baseline:latest": privileged (container "app" must not set securityContext.privileged=true)
```

**2. An ordinary root Pod: allowed by Baseline, with a warning about Restricted**

```bash
kubectl apply -f pod-root-default.yaml -n psa-baseline
# Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false ..., runAsNonRoot != true ..., seccompProfile ...
kubectl get pod root-pod -n psa-baseline                                  # created and running
```

**3. The same Pod is rejected under Restricted**

```bash
kubectl apply -f pod-root-default.yaml -n psa-restricted
# Error ... violates PodSecurity "restricted:latest": ...   (read the list of violated controls)
```

**4. A compliant Pod passes**

```bash
kubectl apply -f pod-restricted-ok.yaml -n psa-restricted                 # created
kubectl get pod restricted-ok -n psa-restricted
```

**5. Predict the impact before enforcing: a server-side dry run**

```bash
kubectl label --dry-run=server --overwrite ns psa-baseline pod-security.kubernetes.io/enforce=restricted
# Warning: existing pods in namespace "psa-baseline" violate the new PodSecurity enforce level "restricted:latest"
#          root-pod: ...
```

Nothing changed. The warning lists which existing Pods would violate the stricter level.

**6. Existing Pods are not touched when you tighten enforcement**

```bash
kubectl label --overwrite ns psa-baseline pod-security.kubernetes.io/enforce=restricted
kubectl get pod root-pod -n psa-baseline                                  # still Running
kubectl delete pod root-pod -n psa-baseline
kubectl apply -f pod-root-default.yaml -n psa-baseline                    # now rejected
```

**Cleanup**

```bash
kubectl delete -f psa-namespaces.yaml
```

## Gotchas

- **Nothing is enforced until you label namespaces.** Check with `kubectl get ns -L pod-security.kubernetes.io/enforce`.
- **`enforce` rejects Pods, not Deployments.** A Deployment is accepted and then its ReplicaSet fails to create Pods. Look at `kubectl describe rs` events.
- **Existing Pods keep running** when you tighten a label. They fail only when recreated.
- **Pin the version** (`-version: v1.30`) if you do not want a Kubernetes upgrade to change the policy.
- **System namespaces** need `privileged`. Do not enforce Baseline or Restricted on `kube-system`.
- Restricted often breaks images that **run as root**. Choose or build non-root images (for example the `nginxinc/nginx-unprivileged` variant of nginx).
- **PodSecurityPolicy no longer exists** (removed in 1.25). Old guides that mention it are outdated.
- PSA cannot express custom rules. Add Kyverno, Gatekeeper or ValidatingAdmissionPolicy for those.

## Check yourself

1. How does Pod Security differ from a `securityContext`?
2. Name the three profiles and what each is for.
3. What are the three PSA modes?
4. How do you enable PSA for a namespace?
5. What does the Restricted profile require that Baseline does not (name three things)?
6. How can you preview what enforcing a profile would break?
7. What happens to existing Pods when you add an `enforce` label?

<details>
<summary>Answers</summary>

1. A securityContext is what a Pod asks for. Pod Security is admission control that makes sure every Pod meets a minimum standard
2. Privileged (no restrictions, for system components), Baseline (blocks known escalations), Restricted (current hardening best practice)
3. `enforce` (reject), `warn` (warning to the user), `audit` (audit log entry)
4. Add `pod-security.kubernetes.io/<mode>: <level>` labels to the namespace
5. Non-root user, `allowPrivilegeEscalation: false`, drop ALL capabilities, a seccomp profile, restricted volume types (any three)
6. A server-side dry run: `kubectl label --dry-run=server --overwrite ns <ns> pod-security.kubernetes.io/enforce=<level>`
7. Nothing: they keep running. The rule applies only to new Pods

</details>

---

[← Security Context](security-context.md) | [README](README.md) | [Network Security →](network-security.md)
