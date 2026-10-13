# Service Accounts

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../labs/08-rbac/)

[README](README.md) | [RBAC →](RBAC/README.md)

---

## Idea

A **ServiceAccount** is the **identity of a Pod**. When a program inside a Pod calls the Kubernetes API (or a cloud service), the ServiceAccount says **who it is**, and [RBAC](RBAC/README.md) decides what it may do.

| | Human user | ServiceAccount |
|---|---|---|
| Represents | A person | A workload (Pod) |
| Stored in Kubernetes? | **No** (comes from the authenticator) | **Yes**, a namespaced object |
| Credential | Certificate, OIDC token, IAM | A **token** mounted into the Pod |
| Full name | `jane` | `system:serviceaccount:<namespace>:<name>` |

## The default ServiceAccount

Every namespace has a ServiceAccount called **`default`**. A Pod that does not name one **uses it**.

```bash
kubectl get serviceaccount -n dev
kubectl describe serviceaccount default -n dev
```

The `default` account has **no useful permissions** unless you grant some. Do not bind roles to it: that would give the permissions to **every Pod** that did not choose an account.

## Creating and using one

```bash
kubectl create serviceaccount app-sa -n dev
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: dev
automountServiceAccountToken: true        # default; set false when the app never calls the API
imagePullSecrets:
  - name: registry-credentials            # optional: Pods using this account can pull private images
```

Use it in a Pod or any controller's template:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo
  namespace: dev
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: curlimages/curl:8.8.0
      command: ["sleep", "3600"]
```

Changing `serviceAccountName` on a running Pod is not possible. Recreate it (or roll the Deployment).

## What gets mounted into the Pod

Kubernetes mounts a projected volume at **`/var/run/secrets/kubernetes.io/serviceaccount/`**:

| File | Content |
|---|---|
| `token` | A signed **JWT** proving the Pod's identity |
| `ca.crt` | The cluster CA, used to trust the API server |
| `namespace` | The Pod's namespace |

```
Pod ──token──► https://kubernetes.default.svc  (API server)
                 authenticates:  system:serviceaccount:dev:app-sa
                 RBAC decides:   allowed or Forbidden
```

Client libraries find all this automatically ("in-cluster config"). For example, in Node.js with `@kubernetes/client-node`, `kc.loadFromCluster()`.

## Tokens: short-lived and bound

Since Kubernetes 1.22 (and by default from 1.24), tokens are **not** permanent Secrets. They are issued through the **TokenRequest API** as **projected tokens** with these properties:

| Property | Meaning |
|---|---|
| **Bound to the Pod** | The token stops being valid when the Pod is deleted |
| **Expiring** | Typically valid for an hour, and the kubelet refreshes it before it expires |
| **Audience-bound** | Valid only for the audience it was issued for (the API server by default) |

You can also request a token yourself (for CI jobs or testing):

```bash
kubectl create token app-sa -n dev                          # short-lived token (default about 1 hour)
kubectl create token app-sa -n dev --duration=15m
kubectl create token app-sa -n dev --audience=vault
```

**Legacy long-lived tokens** are Secrets of type `kubernetes.io/service-account-token`. They never expire and live in etcd. Avoid creating them. Use projected tokens or `kubectl create token`.

### A custom projected token (for another audience)

Some systems (Vault, cloud identity federation) want a token for their own audience:

```yaml
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: vault-token
          mountPath: /var/run/secrets/tokens
  volumes:
    - name: vault-token
      projected:
        sources:
          - serviceAccountToken:
              path: vault-token
              audience: vault
              expirationSeconds: 3600
```

## Give a ServiceAccount permissions (RBAC)

A new ServiceAccount can do almost nothing. Grant exactly what the app needs:

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding app-sa-reads-pods --role=pod-reader --serviceaccount=dev:app-sa -n dev
```

```bash
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:app-sa      # yes
kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:app-sa    # no
```

## Do not mount a token you do not need

If a Pod never calls the Kubernetes API, remove the credential, so a compromised container has nothing to steal:

```yaml
# on the ServiceAccount (applies to all Pods using it)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web
automountServiceAccountToken: false
---
# or on one Pod (overrides the ServiceAccount)
spec:
  automountServiceAccountToken: false
```

## Cloud identity: AWS (IRSA and Pod Identity)

On EKS a ServiceAccount can also be the identity for **AWS** calls, so Pods get temporary IAM credentials **without** long-lived keys and without using the node's role:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: dev
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/s3-reader   # IRSA
```

| Method | How it works |
|---|---|
| **IRSA** (IAM Roles for Service Accounts) | The cluster's OIDC provider is trusted by an IAM role. The Pod's projected token is exchanged for AWS credentials |
| **EKS Pod Identity** | An EKS-managed agent supplies credentials, with the association made through the EKS API |

Either way, the **role is attached to a ServiceAccount**, which is why one ServiceAccount per workload matters. See [IRSA](../15-aws-eks/irsa.md).

## Lab

```bash
cd labs/08-rbac/manifests
kubectl apply -f namespaces.yaml
kubectl apply -f sa-demo.yaml                              # ServiceAccount app-sa and a Pod that uses it
kubectl wait --for=condition=Ready pod/sa-demo -n dev --timeout=60s
```

**1. What is mounted**

```bash
kubectl exec sa-demo -n dev -- ls -l /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec sa-demo -n dev -- cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

**2. Read the token's claims** (do not paste real production tokens into websites)

```bash
TOKEN=$(kubectl exec sa-demo -n dev -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
python3 -c "import sys,base64,json; t=sys.argv[1].split('.')[1]; t+='='*(-len(t)%4); print(json.dumps(json.loads(base64.urlsafe_b64decode(t)),indent=2))" "$TOKEN"
```

Look at `sub` (`system:serviceaccount:dev:app-sa`), `aud`, `exp` (expiry), and the Pod binding under `kubernetes.io`.

**3. Call the API: authenticated, but forbidden**

```bash
kubectl exec sa-demo -n dev -- sh -c '
  T=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  curl -sS --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    -H "Authorization: Bearer $T" \
    https://kubernetes.default.svc/api/v1/namespaces/dev/pods | head -20'
```

The reply is `403 Forbidden` and names `system:serviceaccount:dev:app-sa`: the cluster knows who it is, but RBAC has granted nothing.

**4. Grant a role and retry**

```bash
kubectl apply -f pod-reader-role.yaml                      # Role pod-reader + RoleBinding for app-sa
kubectl exec sa-demo -n dev -- sh -c '
  T=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  curl -sS --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    -H "Authorization: Bearer $T" \
    https://kubernetes.default.svc/api/v1/namespaces/dev/pods | head -20'
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:app-sa      # yes
kubectl auth can-i list pods -n prod --as=system:serviceaccount:dev:app-sa     # no
```

**5. Request a token yourself**

```bash
kubectl create token app-sa -n dev --duration=10m | cut -c1-40
```

**5b. A token for another audience** (a projected token, as in the Vault example above)

```bash
kubectl apply -f projected-token.yaml
kubectl wait --for=condition=Ready pod/projected-token -n dev --timeout=60s
TOKEN2=$(kubectl exec projected-token -n dev -- cat /var/run/secrets/tokens/vault-token)
python3 -c "import sys,base64,json; t=sys.argv[1].split('.')[1]; t+='='*(-len(t)%4); print(json.loads(base64.urlsafe_b64decode(t))['aud'])" "$TOKEN2"     # ['vault']
```

The audience is `vault`, not the API server, so this token is useless against the Kubernetes API. That is the point of audience binding.

**6. Opt out of the token**

```bash
kubectl patch serviceaccount app-sa -n dev -p '{"automountServiceAccountToken":false}'
kubectl delete pod sa-demo -n dev
kubectl apply -f sa-demo.yaml
kubectl exec sa-demo -n dev -- ls /var/run/secrets/kubernetes.io/serviceaccount/    # no such directory
```

**6b. Opt out for a single Pod** (the account itself still mounts tokens elsewhere)

```bash
kubectl patch serviceaccount app-sa -n dev -p '{"automountServiceAccountToken":true}'
kubectl apply -f sa-no-token.yaml                           # Pod-level automountServiceAccountToken: false
kubectl exec sa-no-token -n dev -- ls /var/run/secrets/kubernetes.io/serviceaccount/    # no such directory
```

The Pod setting overrides the ServiceAccount setting.

**Cleanup**

```bash
kubectl delete -f pod-reader-role.yaml -f sa-demo.yaml -f projected-token.yaml -f sa-no-token.yaml --ignore-not-found
```

## Gotchas

- Pods that do not name an account use **`default`**. Never grant permissions to `default`.
- A ServiceAccount is **namespaced**. `app-sa` in `dev` and `app-sa` in `prod` are different identities.
- In RBAC subjects the account needs its **`namespace`**, and `--serviceaccount` takes **`namespace:name`**.
- **Mounted tokens are credentials.** Anyone with a shell in the Pod can use them. Disable them when unused.
- Do not create long-lived token Secrets unless a legacy system forces you to.
- A **403** from a Pod almost always means "no RoleBinding for this ServiceAccount", not a broken token.
- Cloud identity (IRSA, Pod Identity) is per ServiceAccount, so share accounts between workloads only if they truly need the same cloud permissions.

## Check yourself

1. What is a ServiceAccount, and how does it differ from a user?
2. Which ServiceAccount does a Pod use if you do not set one?
3. Which three files are mounted into a Pod, and what do they contain?
4. How are modern ServiceAccount tokens different from the old Secret-based ones?
5. How do you stop a Pod from receiving a token?
6. What is the full username of ServiceAccount `app-sa` in namespace `dev`?

<details>
<summary>Answers</summary>

1. The identity of a Pod, stored in Kubernetes as a namespaced object. A user is a human identity that Kubernetes does not store
2. The namespace's `default` ServiceAccount
3. `token` (a JWT), `ca.crt` (cluster CA) and `namespace`
4. They are bound to the Pod, expire (and are refreshed), and are audience-bound. The old Secrets never expired
5. `automountServiceAccountToken: false` on the ServiceAccount or the Pod
6. `system:serviceaccount:dev:app-sa`

</details>

---

[README](README.md) | [RBAC →](RBAC/README.md)
