# RBAC: Role-Based Access Control

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../../labs/08-rbac/)

[← Service Accounts](../service-accounts.md) | [Module README](../README.md) | [Roles and ClusterRoles →](roles-and-clusterroles.md)

---

## Idea

**RBAC decides what an authenticated identity is allowed to do** in the cluster. Nobody can do anything until a rule grants it.

```
Who are you?      → Authentication     (certificate, token, OIDC, cloud IAM)
What may you do?  → Authorization      ← RBAC lives here
Is the object OK? → Admission control  (quotas, Pod Security, policies)
```

Three properties define RBAC:

| Property | Meaning |
|---|---|
| **Default deny** | With no matching rule, the answer is "forbidden" |
| **Additive only** | You can only **grant** permissions. There are **no deny rules**. Effective permissions are the **union** of all rules that apply |
| **Everything is an API object** | Roles and bindings are ordinary YAML you keep in Git |

## The mental model

Think of an **office building**:

| Building | RBAC |
|---|---|
| A **keycard type** that opens certain doors | **Role** (a set of permissions) |
| **Giving** a person that keycard | **RoleBinding** (who gets which role) |
| The person or badge | **Subject**: User, Group or ServiceAccount |

```
 Subject ──(RoleBinding)──► Role ──(rules)──► verbs on resources
 jane                        pod-reader         get, list on pods
```

A Role says **what is allowed**. A binding says **who gets it**. Neither does anything alone.

## The four objects

| Object | Scope | Contains | Purpose |
|---|---|---|---|
| **Role** | One namespace | Rules | Permissions **inside a namespace** |
| **ClusterRole** | Whole cluster | Rules | Permissions on **cluster-wide** resources, or a **reusable** rule set |
| **RoleBinding** | One namespace | Subjects + a role reference | Grants a role **within that namespace** |
| **ClusterRoleBinding** | Whole cluster | Subjects + a ClusterRole reference | Grants a ClusterRole **everywhere** |

### The four combinations (the table to memorize)

| Role object | Binding object | Result |
|---|---|---|
| **Role** | **RoleBinding** | Permissions in **one namespace** |
| **ClusterRole** | **ClusterRoleBinding** | Permissions in **all namespaces** (and cluster-scoped resources) |
| **ClusterRole** | **RoleBinding** | The ClusterRole's rules, but **only in the binding's namespace** (reuse pattern) |
| Role | ClusterRoleBinding | Not allowed |

The third row is the most useful trick: define a role once (`view`, `edit`) and hand it out per namespace.

## Subjects: who gets the permission

| Kind | What it is | Example |
|---|---|---|
| **User** | A human identity from your authenticator | `jane`, `jane@example.com` |
| **Group** | A set of users from your authenticator | `dev-team`, `system:masters` |
| **ServiceAccount** | An identity for a **Pod** (the only one Kubernetes stores itself) | `app-sa` in namespace `dev` |

Kubernetes has **no User or Group objects**. Those names come from the authentication layer (certificates, OIDC, cloud IAM). See [Users and Groups](users-and-groups.md).

## A rule in one line

```yaml
rules:
  - apiGroups: [""]              # which API group ("" = core)
    resources: ["pods"]          # which resource
    verbs: ["get", "list"]       # which actions
```

Read it as: *"may get and list pods."* Details in [Roles and ClusterRoles](roles-and-clusterroles.md).

## A complete example in 25 lines

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Result: **jane can read Pods in `dev`**, and nothing else anywhere.

```bash
kubectl auth can-i list pods -n dev --as=jane      # yes
kubectl auth can-i list pods -n prod --as=jane     # no
kubectl auth can-i delete pods -n dev --as=jane    # no
```

## Built-in ClusterRoles you will use constantly

| ClusterRole | What it allows |
|---|---|
| **`cluster-admin`** | Everything, everywhere. Treat as "root" |
| **`admin`** | Full control **inside a namespace** (including creating Roles and RoleBindings there) |
| **`edit`** | Read and write most namespaced objects (including Secrets). **Cannot** change roles or bindings |
| **`view`** | Read-only on most namespaced objects. **Cannot read Secrets** or roles |

Hand them out with a RoleBinding to give a person a safe, namespace-limited level of access without writing rules yourself.

## Where to go next

| # | Note | You learn |
|---|---|---|
| 1 | [Roles and ClusterRoles](roles-and-clusterroles.md) | Rule anatomy, verbs, subresources, which kubectl command needs what, built-in and aggregated roles |
| 2 | [RoleBindings and ClusterRoleBindings](rolebindings-and-clusterrolebindings.md) | Subjects, `roleRef`, the four combinations, escalation rules |
| 3 | [Users and Groups](users-and-groups.md) | Where identities come from: certificates, OIDC, EKS access entries; create a real user |
| 4 | [Testing and Debugging](testing-and-debugging.md) | `kubectl auth can-i`, impersonation, reading 403 errors, a fix-the-bug drill |
| 5 | [Best Practices](best-practices.md) | Least privilege, dangerous permissions, CI service accounts, review checklist |

And [ServiceAccounts](../service-accounts.md) for permissions granted to Pods.

## Cheat sheet

```bash
# create
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding read-pods --role=pod-reader --user=jane -n dev
kubectl create rolebinding dev-view --clusterrole=view --group=dev-team -n dev
kubectl create clusterrolebinding ops-nodes --clusterrole=node-reader --group=ops-team
kubectl create rolebinding sa-read --role=pod-reader --serviceaccount=dev:app-sa -n dev

# inspect
kubectl get roles,rolebindings -n dev
kubectl get clusterroles,clusterrolebindings
kubectl describe role pod-reader -n dev
kubectl describe clusterrole view
kubectl get rolebindings,clusterrolebindings -A -o wide

# test
kubectl auth can-i list pods -n dev --as=jane
kubectl auth can-i --list -n dev --as=jane
kubectl auth whoami
```

## Quiz

1. What does RBAC add that authentication does not?
2. Why can RBAC never "deny" something?
3. What is the difference between a Role and a RoleBinding?
4. Which two combinations of role and binding are valid for namespace-limited access?
5. What happens with a ClusterRole and a RoleBinding?
6. Does Kubernetes store User objects?
7. What are the four built-in roles for people, from most to least powerful?
8. Why can `view` not read Secrets?

<details>
<summary>Answers</summary>

1. Authorization: it decides what an identified user may do
2. Rules only grant. Effective permissions are the union of all rules, and anything unlisted is denied by default
3. A Role defines permissions. A RoleBinding gives a role to subjects
4. Role + RoleBinding, and ClusterRole + RoleBinding (reuse)
5. The ClusterRole's rules apply only in the binding's namespace
6. No. Users and groups come from the authenticator. Only ServiceAccounts are stored
7. `cluster-admin`, `admin`, `edit`, `view`
8. Secret contents are sensitive, so `view` excludes them to stay safe to hand out widely

</details>

---

[← Service Accounts](../service-accounts.md) | [Module README](../README.md) | [Roles and ClusterRoles →](roles-and-clusterroles.md)
