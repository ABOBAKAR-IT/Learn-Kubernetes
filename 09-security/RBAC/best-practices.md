# RBAC Best Practices

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../../labs/08-rbac/)

[← Testing and Debugging](testing-and-debugging.md) | [Module README](../README.md)

---

## Idea

RBAC is easy to make **work** and easy to make **too generous**. The goal is **least privilege**: every person and workload gets exactly the access it needs, in the narrowest scope, for as long as it needs it.

## The principles

| Principle | In practice |
|---|---|
| **Least privilege** | Start from nothing and add what is needed. Never start from `cluster-admin` and remove |
| **Narrowest scope** | Prefer **Role + RoleBinding** (one namespace) over cluster-wide |
| **Bind to groups, not people** | Access follows team membership in your identity provider |
| **One identity per workload** | A dedicated ServiceAccount per app, never the shared `default` one |
| **Avoid wildcards** | `*` in verbs, resources or groups also grants future resources |
| **Short-lived credentials** | Short certificate expiry, OIDC tokens, `kubectl create token --duration` |
| **Everything in Git** | RBAC as YAML, reviewed through pull requests |
| **Audit regularly** | Who has `cluster-admin`? Which roles use wildcards? |

## Dangerous permissions

Some permissions look harmless but allow **privilege escalation**:

| Permission | Why it is dangerous |
|---|---|
| `get` / `list` / `watch` on **secrets** | Reads every secret value. **`list` alone is enough** |
| **`create` on `pods`** (in a namespace) | A new Pod can **mount any Secret** and run **as any ServiceAccount** there, then print or send them |
| `create` on **`pods/exec`**, `pods/attach` | A shell inside running Pods exposes their secrets and tokens |
| `create` on **`serviceaccounts/token`** | Mint a token for any ServiceAccount, then act as it |
| **`impersonate`** | Act as any user, group or ServiceAccount, even `system:masters` |
| **`escalate`**, **`bind`** | Grant yourself or others more than you hold |
| Create or edit **Roles, RoleBindings, ClusterRoles, ClusterRoleBindings** | Change RBAC itself |
| `create` / `patch` on **Deployments, DaemonSets, StatefulSets, CronJobs, Jobs** | Indirectly creates Pods with any spec, same as creating Pods |
| `nodes/proxy` | Reaches the kubelet API on nodes |
| **`approve` on CSRs** (and `create` CSR) | Issue client certificates, including powerful ones |
| **`*` on `*`** (cluster-admin) | Everything |
| Membership in **`system:masters`** | Bypasses RBAC entirely |

**Pod creation is the classic escalation.** Whoever can create Pods in a namespace can read every Secret in it. This is why [Pod Security](../pod-security.md) and namespace separation matter, and why "edit in prod" should be rare.

## Practical rules

1. **Do not hand out `cluster-admin`.** Use it for a small platform group, ideally as break-glass. Daily work uses `admin`, `edit` or `view` in specific namespaces.
2. **Use the built-in roles through RoleBindings**: `edit` for developers in dev and staging, `view` in prod.
3. **Treat `edit` as powerful.** It reads Secrets and creates Pods.
4. **Do not grant on `secrets`** unless a workload truly needs a specific Secret. Use `resourceNames` with `get` for one named Secret.
5. **Avoid binding to broad groups:** `system:authenticated`, `system:serviceaccounts`, `system:unauthenticated`.
6. **Disable token mounting** for Pods that never call the API (see [ServiceAccounts](../service-accounts.md)).
7. **Remove stale bindings** when roles, people or services go away.
8. **Review wildcards** and `cluster-admin` bindings on a schedule.
9. **Scope CI/CD tightly** (below).
10. **Keep a tested break-glass admin** that does not depend on the identity provider.

## CI/CD: a deployer, not a cluster admin

A pipeline (GitLab CI, GitHub Actions, Argo CD) should deploy to **one namespace** with only the verbs it uses:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: dev
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: dev
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

Try it (the same manifest is in `labs/08-rbac/manifests/deployer.yaml`):

```bash
cd labs/08-rbac/manifests
kubectl apply -f namespaces.yaml -f deployer.yaml
SA=system:serviceaccount:dev:ci-deployer
kubectl auth can-i create deployments -n dev  --as=$SA          # yes
kubectl auth can-i delete deployments -n dev  --as=$SA          # no: it can roll out, not delete
kubectl auth can-i get secrets -n dev         --as=$SA          # no
kubectl auth can-i create deployments -n prod --as=$SA          # no: one namespace only
kubectl auth can-i create clusterrolebindings --as=$SA          # no: it cannot change RBAC
kubectl delete -f deployer.yaml
```

- **One ServiceAccount per environment**, so a leaked dev credential cannot touch prod.
- Prefer **short-lived tokens** (`kubectl create token ci-deployer -n dev --duration=1h`) or **OIDC federation** from the CI system to your cloud identity, instead of a long-lived kubeconfig stored as a CI variable.
- **Never** store a `cluster-admin` kubeconfig in CI variables.
- On EKS, a CI role can be given an **access entry** scoped to a namespace (see [Users and Groups](users-and-groups.md)).
- If the pipeline needs to create RBAC objects or CRDs, give that to a **separate, more protected** job.

## A starter role set for a team

| Who | Role | Binding | Where |
|---|---|---|---|
| Platform admins (few) | `cluster-admin` | ClusterRoleBinding to a group | Cluster, break-glass |
| Namespace owners | `admin` | RoleBinding | Their namespaces |
| Developers | `edit` | RoleBinding | dev, staging |
| Developers | `view` | RoleBinding | prod |
| Support, auditors | `view` | RoleBinding | Namespaces they cover |
| CI deployers | custom `deployer` | RoleBinding to a ServiceAccount | One namespace each |
| Monitoring | A read-only ClusterRole (Pods, nodes, metrics) | ClusterRoleBinding | Cluster |

## Auditing commands

```bash
# who is cluster-admin?
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | "\(.metadata.name): \(.subjects // [] | map("\(.kind)/\(.name)") | join(", "))"'

# roles that use wildcards
kubectl get clusterroles,roles -A -o json | jq -r '.items[] | select(.rules[]? | ((.verbs // [])|index("*")) or ((.resources // [])|index("*"))) | "\(.kind) \(.metadata.namespace // "-")/\(.metadata.name)"'

# who can read secrets? (needs the kubectl-who-can plugin)
kubectl who-can get secrets -A

# what can the default ServiceAccount do in a namespace?
kubectl auth can-i --list --as=system:serviceaccount:dev:default -n dev
```

## Lab: see why Pod creation is an escalation

Setup, with the lab manifests:

```bash
cd labs/08-rbac/manifests
kubectl apply -f namespaces.yaml                 # namespaces dev and prod, and a Secret in dev
kubectl apply -f pod-creator.yaml                # jane may create Pods and read their logs in dev. Nothing else
```

**1. Jane cannot read the Secret directly**

```bash
kubectl auth can-i get secret/db-secret -n dev --as=jane            # no
kubectl get secret db-secret -n dev --as=jane                       # Forbidden
```

**2. She can create a Pod, and the Pod can use the Secret**

```bash
kubectl apply -f secret-stealer-pod.yaml --as=jane                  # allowed: she may create Pods
kubectl logs steal -n dev --as=jane                                 # prints: the password is: S3cr3t-demo-pass
```

Jane never had permission to read Secrets, yet she got the value through a Pod.

**3. The fixes**

- Do not give `create pods` to anyone who must not read that namespace's secrets.
- Enforce [Pod Security](../pod-security.md) and use admission policies to limit what Pods may reference.
- Separate sensitive workloads into **their own namespace**.

**Cleanup**

```bash
kubectl delete pod steal -n dev
kubectl delete -f pod-creator.yaml
```

## Gotchas

- **Convenience grants stay forever.** Time-box exceptions and review them.
- `edit` and `admin` can read Secrets and create Pods, so they are **not** "safe" roles.
- A permission on **Deployments** is effectively a permission to create Pods.
- Granting `get` on a Secret by name is fine, `list` on all Secrets is not.
- RBAC controls **API access**. It does not limit what a running container can do (see [security context](../security-context.md)) or who it can reach on the network (see [network security](../network-security.md)).
- Removing a person's group membership removes access only if **every** path (groups, direct bindings, cloud IAM) is cleaned up.

## Review checklist

- [ ] Nobody has `cluster-admin` for daily work, and the group is small
- [ ] No bindings to `system:authenticated`, `system:unauthenticated` or `system:serviceaccounts` with real permissions
- [ ] No wildcard roles, except system ones
- [ ] People are bound through **groups**
- [ ] Each workload uses its **own** ServiceAccount
- [ ] CI/CD identities are namespace-scoped and short-lived
- [ ] `secrets`, `pods/exec`, `serviceaccounts/token`, `impersonate`, `escalate`, `bind` are granted rarely and deliberately
- [ ] RBAC is in Git and reviewed
- [ ] Audit logging is enabled
- [ ] A break-glass path exists and is tested

## Check yourself

1. What is least privilege, in one sentence?
2. Why is the ability to create Pods in a namespace effectively access to its Secrets?
3. Name four permissions that allow privilege escalation.
4. How should a CI pipeline get cluster access?
5. Why do `edit` and `admin` deserve caution?
6. What should you do when a role is no longer needed?

<details>
<summary>Answers</summary>

1. Give each identity only the permissions it needs, in the narrowest scope, for as long as needed
2. A new Pod can mount any Secret in the namespace (or run as any ServiceAccount) and reveal the values
3. For example: `create pods`, `pods/exec`, `serviceaccounts/token`, `impersonate`, `escalate`, `bind`, editing RBAC objects, `list` on secrets
4. With a namespace-scoped ServiceAccount (or federated identity) using short-lived credentials, never a stored cluster-admin kubeconfig
5. They can read Secrets and create Pods, which allows reading secrets and running workloads
6. Delete the role and its bindings, since deleting a role leaves its bindings behind

</details>

---

[← Testing and Debugging](testing-and-debugging.md) | [Module README](../README.md)
