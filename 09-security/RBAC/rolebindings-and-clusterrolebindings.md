# RoleBindings and ClusterRoleBindings

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../../labs/08-rbac/)

[← Roles and ClusterRoles](roles-and-clusterroles.md) | [Module README](../README.md) | [Users and Groups →](users-and-groups.md)

---

## Idea

A **binding** connects **who** (subjects) to **what** (a role). It is the object that actually **grants** permissions.

```
RoleBinding / ClusterRoleBinding
├── subjects:  who gets it      (User, Group, ServiceAccount)
└── roleRef:   which role       (Role or ClusterRole)
```

| Binding | Scope of the grant | May reference |
|---|---|---|
| **RoleBinding** | **One namespace** (its own) | A Role in that namespace, **or** any ClusterRole |
| **ClusterRoleBinding** | **The whole cluster** | Only a ClusterRole |

## YAML explained

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev                         # where the permission applies
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: app-sa
    namespace: dev                       # ServiceAccounts need a namespace, and no apiGroup
roleRef:
  kind: Role                             # Role or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

| Field | Meaning |
|---|---|
| `subjects[]` | One or more identities that receive the role |
| `subjects[].kind` | `User`, `Group` or `ServiceAccount` |
| `subjects[].apiGroup` | `rbac.authorization.k8s.io` for User and Group. **Omit for ServiceAccount** |
| `subjects[].namespace` | **Required for ServiceAccount** (the namespace the account lives in) |
| `roleRef` | The role to grant. It **cannot be changed** after creation |

## The four combinations

**1. Role + RoleBinding**: permissions in **one namespace**

```yaml
kind: RoleBinding
metadata: { name: read-pods, namespace: dev }
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
```

**2. ClusterRole + ClusterRoleBinding**: permissions in **all namespaces** and on cluster-scoped resources

```yaml
kind: ClusterRoleBinding
metadata: { name: ops-read-nodes }
subjects:
  - { kind: Group, name: ops-team, apiGroup: rbac.authorization.k8s.io }
roleRef: { kind: ClusterRole, name: node-reader, apiGroup: rbac.authorization.k8s.io }
```

**3. ClusterRole + RoleBinding**: the ClusterRole's rules, but **only in the binding's namespace**. This is the **reuse pattern**:

```yaml
kind: RoleBinding
metadata: { name: dev-team-view, namespace: dev }
subjects:
  - { kind: Group, name: dev-team, apiGroup: rbac.authorization.k8s.io }
roleRef: { kind: ClusterRole, name: view, apiGroup: rbac.authorization.k8s.io }
```

`dev-team` can view everything in `dev` and nothing in `prod`, using the built-in `view` role without copying its rules.

**4. Role + ClusterRoleBinding**: **not possible**. A ClusterRoleBinding can only reference a ClusterRole.

```
                    RoleBinding               ClusterRoleBinding
   Role           ✔ one namespace                  ✘
   ClusterRole    ✔ one namespace (reuse)    ✔ every namespace
```

Compare what `view` gives with each:

| Binding | Where the person can read |
|---|---|
| RoleBinding in `dev` → `view` | Only `dev` |
| ClusterRoleBinding → `view` | Every namespace |

## Subjects in detail

### Users and groups

Names are exactly what the **authenticator** reports. They are just strings:

| Source | Typical username | Typical groups |
|---|---|---|
| Client certificate | the certificate's `CN` | the certificate's `O` values |
| OIDC | a claim such as `email`, often with a prefix like `oidc:` | the groups claim |
| EKS | the mapped IAM user or role identity | Kubernetes groups you assign to the access entry |

Check what the cluster sees: `kubectl auth whoami`.

### ServiceAccounts

A ServiceAccount's full identity is `system:serviceaccount:<namespace>:<name>`. Pods run as one (see [ServiceAccounts](../service-accounts.md)).

```bash
kubectl create rolebinding sa-read --role=pod-reader --serviceaccount=dev:app-sa -n dev     # note: namespace:name
```

### Special groups

| Group | Contains |
|---|---|
| `system:authenticated` | Everyone with valid credentials |
| `system:unauthenticated` | **Anonymous** requests (never grant useful permissions here) |
| `system:serviceaccounts` | **All** ServiceAccounts in the cluster |
| `system:serviceaccounts:<ns>` | All ServiceAccounts in one namespace |
| `system:masters` | **Super-admins**: bypasses RBAC completely, cannot be restricted |
| `system:nodes` | Kubelets |

Granting a role to `system:authenticated` or `system:serviceaccounts` hands it to a very large audience.

## Create bindings from the command line

```bash
kubectl create rolebinding read-pods --role=pod-reader --user=jane -n dev
kubectl create rolebinding dev-view --clusterrole=view --group=dev-team -n dev
kubectl create rolebinding sa-read --role=pod-reader --serviceaccount=dev:app-sa -n dev
kubectl create clusterrolebinding ops-nodes --clusterrole=node-reader --group=ops-team

kubectl create rolebinding dev-view --clusterrole=view --group=dev-team -n dev --dry-run=client -o yaml
```

## Rules to remember

| Rule | Detail |
|---|---|
| **Effective permission = union** | A subject gets everything from **every** binding that matches it |
| **`roleRef` is immutable** | To switch roles, **delete and recreate** the binding |
| **Subjects can be edited** | Add or remove people with `kubectl edit` or `apply` |
| **A binding can point to a role that does not exist yet** | It stays inert until the role appears |
| **Deleting a role does not delete its bindings** | They keep dangling. Clean up both |
| **No privilege escalation** | You cannot create or edit a role granting permissions you do not hold, nor bind a role you do not hold, unless you have the `escalate` and `bind` verbs |

Idempotent updates of RBAC manifests:

```bash
kubectl auth reconcile -f rbac.yaml        # creates or updates, recreating bindings when roleRef changes
```

## Inspecting

```bash
kubectl get rolebindings -n dev
kubectl get clusterrolebindings
kubectl get rolebindings,clusterrolebindings -A -o wide              # includes role and subjects
kubectl describe rolebinding read-pods -n dev
kubectl describe clusterrolebinding cluster-admin
```

Default bindings you will see in every cluster:

| ClusterRoleBinding | Grants | To |
|---|---|---|
| `cluster-admin` | `cluster-admin` | `system:masters` |
| `system:basic-user` | `system:basic-user` (lets people ask what they can do) | `system:authenticated` |
| `system:discovery` | API discovery | `system:authenticated` |
| `system:public-info-viewer` | Health and version | `system:authenticated` and `system:unauthenticated` |

## Gotchas

- **A ServiceAccount subject needs its `namespace`**, and no `apiGroup`. User and Group subjects need the `apiGroup` and no namespace.
- A **typo in the user or group name** produces a valid binding that matches nobody, with no error. Check with `kubectl auth can-i --as`.
- A RoleBinding in namespace **X** grants permissions **in X only**, even when it references a ClusterRole.
- Binding the **wrong kind** (a Role named like a ClusterRole) fails or does nothing, so check `roleRef.kind`.
- Prefer binding to **groups**, not individuals, so access follows team membership.
- Bindings to `system:unauthenticated` or `system:authenticated` are almost always a mistake.
- Changing `roleRef` with `kubectl apply` fails with "cannot change roleRef". Delete and recreate, or use `kubectl auth reconcile`.

## Check yourself

1. What are the two parts of a binding?
2. What does a RoleBinding that references a ClusterRole do?
3. Why can a Role not be used in a ClusterRoleBinding?
4. What extra field does a ServiceAccount subject need?
5. How do you change the role a binding points to?
6. What happens when a user matches two bindings?
7. Why is `system:masters` special?

<details>
<summary>Answers</summary>

1. `subjects` (who) and `roleRef` (which role)
2. It grants the ClusterRole's rules only inside the RoleBinding's namespace
3. A ClusterRoleBinding applies cluster-wide and can only reference a ClusterRole, because a Role belongs to one namespace
4. `namespace` (and no `apiGroup`)
5. Delete and recreate the binding (`roleRef` is immutable), or use `kubectl auth reconcile`
6. The user gets the union of both roles' permissions
7. Members bypass RBAC and are full cluster administrators that RBAC cannot restrict

</details>

---

[← Roles and ClusterRoles](roles-and-clusterroles.md) | [Module README](../README.md) | [Users and Groups →](users-and-groups.md)
