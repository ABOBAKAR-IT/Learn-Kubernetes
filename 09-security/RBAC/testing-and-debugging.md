# Testing and Debugging RBAC

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../../labs/08-rbac/)

[← Users and Groups](users-and-groups.md) | [Module README](../README.md) | [Best Practices →](best-practices.md)

---

## Idea

RBAC mistakes are silent: a typo in a name creates a valid object that matches nobody. So **test every grant** the same way you test code. Kubernetes gives you two tools for it: `kubectl auth can-i` and **impersonation**.

## `kubectl auth can-i`

Asks the API server: "would this be allowed?"

```bash
kubectl auth can-i create deployments                      # as you, in the current namespace
kubectl auth can-i create deployments -n dev
kubectl auth can-i list pods --all-namespaces
kubectl auth can-i get pods --subresource=log -n dev       # subresources
kubectl auth can-i get secret/db-secret -n dev             # one named object
kubectl auth can-i '*' '*'                                 # are you cluster-admin?
kubectl auth can-i --list -n dev                           # EVERYTHING you may do in dev
```

Answers are `yes` or `no` (and `--quiet` returns only an exit code, useful in scripts).

### Test as someone else (impersonation)

```bash
kubectl auth can-i list pods -n dev --as=jane
kubectl auth can-i list pods -n dev --as=jane --as-group=dev-team
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:app-sa
kubectl auth can-i --list -n dev --as=jane --as-group=dev-team
```

| Flag | Meaning |
|---|---|
| `--as=<user>` | Act as that username |
| `--as-group=<group>` | Add a group (needs `--as`; repeat for several) |
| `--as=system:serviceaccount:<ns>:<name>` | Act as a ServiceAccount |

You can use **any** kubectl command with `--as`, not just `can-i`:

```bash
kubectl get pods -n dev --as=jane --as-group=dev-team      # the real error message, as jane would see it
```

Impersonation itself needs the **`impersonate`** verb, and it is a powerful permission (see [best practices](best-practices.md)). As a cluster admin you have it.

### See who you are

```bash
kubectl auth whoami
kubectl --as=jane --as-group=dev-team auth whoami
```

## Reading the error

```
Error from server (Forbidden): pods is forbidden:
User "jane" cannot list resource "pods" in API group "" in the namespace "dev"
```

Every part tells you what to grant:

| Piece | Means | Fix |
|---|---|---|
| `User "jane"` | The identity (and groups) the API server saw | Is it the name you bound? |
| `cannot list` | The **verb** | Add `list` |
| `resource "pods"` | The **resource** (or `pods/log`, `deployments/scale`) | Add that resource or subresource |
| `API group ""` | The **API group** | Use `""` for core, `"apps"` for Deployments... |
| `in the namespace "dev"` | **Where** | Bind in that namespace, or use a ClusterRoleBinding |

A request with no namespace in the message is cluster-scoped (nodes, namespaces, PersistentVolumes) and needs a **ClusterRole** and **ClusterRoleBinding**.

### 401 or 403?

| Code | Message | Meaning | Look at |
|---|---|---|---|
| **401** | `You must be logged in to the server (Unauthorized)` | **Authentication** failed: the cluster does not know who you are | Kubeconfig, expired token or certificate, IAM or OIDC setup |
| **403** | `Forbidden` | Authenticated, but **not authorized** | RBAC (this module) |

## A debugging flow for "Forbidden"

```
1. Read the message:  who? which verb? which resource and group? which namespace?
2. kubectl auth can-i <verb> <resource> -n <ns> --as=<user> [--as-group=<group>]   reproduce it
3. Is the identity right?       kubectl auth whoami      (username spelling, group names, prefixes)
4. Does a binding match it?     list bindings that mention the user or group
5. Does the role grant it?      kubectl describe role/clusterrole <name>
6. Is the scope right?          namespace of the Role and RoleBinding vs the namespace you use
7. Fix, then re-run step 2.
```

### Find the bindings that apply to a subject

```bash
kubectl get rolebindings,clusterrolebindings -A -o json | jq -r '
  .items[]
  | select(.subjects[]? | select((.kind=="User" and .name=="jane") or (.kind=="Group" and .name=="dev-team")))
  | "\(.kind)  \(.metadata.namespace // "cluster-wide")/\(.metadata.name)  ->  \(.roleRef.kind)/\(.roleRef.name)"'
```

### Find who can do something

The built-in tools only answer "can **this** subject do X". To ask "**who** can do X", use a plugin:

| Tool | Answers |
|---|---|
| `kubectl who-can get secrets -n dev` (kubectl-who-can) | Which subjects can do that |
| `kubectl rakkess` | A matrix of resources and verbs for a subject |
| `rbac-lookup` | Which roles a subject has |
| `audit2rbac` | Generates a minimal role from **audit log** denials |

Also enable **audit logging** (on EKS, send the `audit` control-plane logs to CloudWatch) to see exactly which requests were denied and why.

## Applying RBAC safely

```bash
kubectl auth reconcile -f rbac.yaml              # create or update RBAC objects idempotently
kubectl apply -f rbac.yaml --dry-run=server      # validate without changing anything
kubectl diff -f rbac.yaml                        # see what would change
```

Keep RBAC YAML in Git and review changes like code.

## Common problems

| Symptom | Likely cause | Fix |
|---|---|---|
| Can `get pod` but `kubectl get pods` fails | Missing `list` | Add `list` |
| `kubectl logs` forbidden | Rule has `pods` but not `pods/log` | Add `pods/log` |
| `kubectl exec` forbidden | Missing `create` on `pods/exec` | Add it (and treat it as sensitive) |
| `kubectl scale` forbidden | Missing `deployments/scale` | Add the subresource |
| Deployment rule never matches | `apiGroups: [""]` instead of `["apps"]` | Use the right group |
| `pod` / `deployment` rule never matches | Singular names | Use plural |
| Works in `dev`, forbidden in `prod` | Role and binding live in `dev` only | Bind in `prod` too |
| Group has access but members do not | Group name typo, or the authenticator does not send it | Check `kubectl auth whoami` |
| ServiceAccount gets 403 | Binding subject has the wrong `namespace`, or you bound the wrong ServiceAccount | Check `system:serviceaccount:<ns>:<name>` |
| Node or PV rule in a Role does nothing | Cluster-scoped resource needs a ClusterRole + ClusterRoleBinding | Use cluster-level objects |
| `cannot change roleRef` | `roleRef` is immutable | Delete and recreate the binding |
| Your edit works for you but not the user | You tested as admin | Test with `--as` |

## Drill: fix four broken setups

`labs/08-rbac/manifests/broken-rbac.yaml` has four setups, each with **one** mistake:

```bash
cd labs/08-rbac/manifests
kubectl apply -f namespaces.yaml
kubectl apply -f broken-rbac.yaml

kubectl auth can-i list deployments -n dev --as=alice                                # expected: yes, got: ?
kubectl auth can-i get pods --subresource=log -n dev --as=bob                        # expected: yes
kubectl auth can-i list pods -n dev --as=carol                                       # expected: yes
kubectl auth can-i list pods -n dev --as=dave --as-group=qa-team                     # expected: yes
```

Find each fault using `kubectl describe` and the table above, fix the YAML and re-test.

<details>
<summary>Solutions</summary>

1. **alice**: the Role uses `apiGroups: [""]` for `deployments`. Deployments are in the `apps` group, so change it to `apiGroups: ["apps"]`.
2. **bob**: the Role grants `pods` only. `kubectl logs` needs `get` on **`pods/log`**, so add `pods/log` to `resources`.
3. **carol**: the Role and RoleBinding are in **`prod`**, but she works in `dev`. Create them in `dev` (or add a binding there).
4. **dave / qa-team**: the binding names the group **`qa-teem`** (typo). Change it to `qa-team`.

</details>

**Cleanup**

```bash
kubectl delete -f broken-rbac.yaml
```

## Gotchas

- **Test with `--as`, not as admin.** Your own admin rights hide every mistake.
- **`kubectl auth can-i` reports what RBAC would decide.** Admission policies (for example Pod Security) can still reject the request afterwards.
- A binding with a typo is **valid YAML and valid API**, so nothing warns you.
- `can-i --list` for a user shows rules from **all** their bindings, which is the quickest "what can they do" check.
- The `no` answer does not say **why**. Use the debugging flow above.
- Impersonation needs permission itself and leaves its own audit trail.

## Check yourself

1. Which command lists everything a user can do in a namespace?
2. What does `--as-group` do, and what must you pass with it?
3. How do you tell a 401 from a 403, and what does each mean?
4. A user can `get` a Pod but `kubectl get pods` fails. Why?
5. Why do you test with `--as` instead of as yourself?
6. Name three tools that answer "who can do X".

<details>
<summary>Answers</summary>

1. `kubectl auth can-i --list -n <ns> --as=<user>` (add `--as-group` for groups)
2. It adds a group to the impersonated identity, and you must also pass `--as`
3. 401 Unauthorized: authentication failed, the cluster does not know you. 403 Forbidden: you are known, but RBAC does not allow it
4. `get pods` (one Pod) needs `get`, while listing needs the `list` verb, which is missing
5. As an administrator you can do everything, which hides missing permissions
6. For example kubectl-who-can, rakkess, rbac-lookup (also audit2rbac to generate roles from audit logs)

</details>

---

[← Users and Groups](users-and-groups.md) | [Module README](../README.md) | [Best Practices →](best-practices.md)
