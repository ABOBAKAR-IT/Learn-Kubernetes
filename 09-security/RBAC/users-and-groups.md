# Users and Groups

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../../labs/08-rbac/)

[← Bindings](rolebindings-and-clusterrolebindings.md) | [Module README](../README.md) | [Testing and Debugging →](testing-and-debugging.md)

---

## Idea

**Kubernetes has no User or Group objects.** You cannot run `kubectl create user`. Identities come from **authentication**: the API server asks "who is this?" and a plugin answers with a **username** and a list of **groups**. RBAC then works with those two strings.

```
credential ─► authenticator ─► username "jane", groups ["dev-team", "system:authenticated"]
                                          │
                                          ▼
                               RBAC: which bindings mention jane or dev-team?
```

The only identities Kubernetes **stores** are [ServiceAccounts](../service-accounts.md), which are for Pods.

## Ways to authenticate

| Method | Identity comes from | Good for |
|---|---|---|
| **Client certificates** | Certificate: `CN` = username, `O` = groups | Admin tools, learning, break-glass |
| **ServiceAccount tokens** | A signed token | Pods and automation |
| **OIDC** | An identity provider (Okta, Azure AD, Google, Keycloak) | **Humans** in real organizations |
| **Webhook token auth** | A custom service | Special integrations |
| **Cloud IAM** | AWS IAM (EKS), Google (GKE), Azure AD (AKS) | Managed clusters |
| ~~Static token or password files~~ | A file on the API server | Avoid |

Anyone who is not authenticated is treated as `system:anonymous` in `system:unauthenticated`, which should have **no** permissions.

## Check who you are

```bash
kubectl auth whoami                       # username, groups, extra info
kubectl config view --minify              # which credentials this context uses
```

## Create a real user with a certificate

This shows exactly how a username and group are born. The certificate's **CN** becomes the username and its **O** becomes the group.

**1. Key and signing request**

```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=dev-team"
```

**2. Submit it to the cluster**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(base64 < jane.csr | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400            # 1 day
  usages:
    - client auth
EOF
kubectl get csr jane                  # Pending
```

**3. Approve and fetch the certificate**

```bash
kubectl certificate approve jane
kubectl get csr jane                  # Approved,Issued
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

**4. Build a kubeconfig entry**

```bash
CLUSTER=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
kubectl config set-credentials jane --client-certificate=jane.crt --client-key=jane.key --embed-certs=true
kubectl config set-context jane-ctx --cluster=$CLUSTER --user=jane --namespace=dev
```

**5. Authenticated, but not authorized**

```bash
kubectl --context=jane-ctx auth whoami                # Username jane, Groups dev-team, system:authenticated
kubectl --context=jane-ctx get pods -n dev            # Error: Forbidden (jane may do nothing yet)
```

**6. Grant a role to her group, and try again**

```bash
kubectl create rolebinding dev-team-view --clusterrole=view --group=dev-team -n dev
kubectl --context=jane-ctx get pods -n dev            # works
kubectl --context=jane-ctx get pods -n prod           # Forbidden: only dev
```

You granted access to the **group**, so anyone presenting a certificate with `O=dev-team` gets it.

### Certificate cautions

- Kubernetes **cannot revoke** a client certificate (no revocation list). It is valid until it expires, so use a **short expiry**.
- Anyone who obtains the key **is** that user.
- For people, prefer **OIDC** or cloud IAM. Use certificates only for simple, short-lived cases.
- A CSR approver is powerful: `approve` on CSRs lets someone mint credentials (even for `system:masters`).

## OIDC: the right way for humans

With OIDC the cluster trusts tokens signed by your identity provider (Okta, Azure AD, Google Workspace, Keycloak...):

```
user logs in at the IdP ─► gets an ID token ─► kubectl sends it ─► API server validates it
                                                    username = a claim (email), groups = a claim
```

You configure which claim becomes the username and which becomes the groups, often with a prefix (for example `oidc:`), so bindings read `oidc:dev-team`. Benefits: central user management, instant removal when someone leaves, MFA, and short-lived tokens. On managed clusters you configure OIDC through the provider instead of API server flags.

## Amazon EKS: IAM identities

EKS authenticates with AWS IAM. `aws eks update-kubeconfig` sets up kubectl to fetch a token for your IAM identity. Then you map IAM principals to Kubernetes permissions.

**Access entries (the recommended method)** connect an IAM user or role to the cluster through the EKS API:

```bash
# let EKS read access entries (new clusters usually have this already)
aws eks update-cluster-config --name my-cluster --access-config authenticationMode=API_AND_CONFIG_MAP

# an IAM role becomes member of a Kubernetes group
aws eks create-access-entry --cluster-name my-cluster \
  --principal-arn arn:aws:iam::111122223333:role/DevRole \
  --kubernetes-groups dev-team

# or use an AWS-managed access policy, limited to a namespace
aws eks associate-access-policy --cluster-name my-cluster \
  --principal-arn arn:aws:iam::111122223333:role/DevRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=dev
```

| Choice | Meaning |
|---|---|
| `--kubernetes-groups dev-team` | The IAM role appears as that Kubernetes group, so **your own RBAC** (`dev-team` bindings) applies |
| Managed access policies | AWS-provided levels that resemble `cluster-admin`, `admin`, `edit` and `view` (`AmazonEKSClusterAdminPolicy`, `AmazonEKSAdminPolicy`, `AmazonEKSEditPolicy`, `AmazonEKSViewPolicy`), optionally scoped to namespaces |
| Authentication modes | `CONFIG_MAP` (legacy only), `API_AND_CONFIG_MAP` (both, access entries win), `API` (access entries only) |

The older **`aws-auth` ConfigMap** in `kube-system` did the same mapping inside the cluster. It is **deprecated**, and AWS recommends migrating to access entries, which are managed from outside the cluster and recorded in CloudTrail. Do not remove existing `aws-auth` entries (for example those created for node groups) before equivalent access entries exist.

With Terraform, use the `aws_eks_access_entry` and `aws_eks_access_policy_association` resources. Check the AWS and provider documentation for the current options.

## A group strategy that scales

| Group | Granted | Where |
|---|---|---|
| `platform-admins` | `cluster-admin` (few people, ideally break-glass) | ClusterRoleBinding |
| `team-a-devs` | `edit` | RoleBinding in `team-a-dev`, `team-a-staging` |
| `team-a-devs` | `view` | RoleBinding in `team-a-prod` |
| `support` | `view` | RoleBinding in the namespaces they help with |
| `ops-team` | A ClusterRole to read nodes | ClusterRoleBinding |

Map **people to groups in the identity provider** (or IAM), and **groups to roles in RBAC**. When someone changes teams you edit the IdP, not dozens of bindings.

## Lab

```bash
# Part of the full lab in labs/08-rbac. In short:
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=dev-team"
# apply the CSR, approve, build the kubeconfig (above), then:
kubectl create namespace dev
kubectl create rolebinding dev-team-view --clusterrole=view --group=dev-team -n dev
kubectl --context=jane-ctx get pods -n dev
```

Delete the test identity afterwards:

```bash
kubectl delete csr jane
kubectl config delete-user jane ; kubectl config delete-context jane-ctx
rm -f jane.key jane.csr jane.crt
```

## Gotchas

- **There is no `kubectl create user`.** If a name shows in a binding, it only matters if some authenticator can produce it.
- A **misspelled username or group** silently matches nobody.
- **Client certificates cannot be revoked**, so keep them short-lived and rare.
- A certificate's **O values are groups**: `O=system:masters` gives full cluster control. Never issue it casually.
- With OIDC, names often carry a **prefix**, and bindings must use the prefixed name.
- On EKS, creating the access entry is not enough: it needs a **group with RBAC bindings**, or an **access policy**, to grant anything.
- Keep a **break-glass** administrator path that does not depend on your identity provider.

## Check yourself

1. Where do Kubernetes usernames and groups come from?
2. In a client certificate, which field is the username and which is the group?
3. Why is it a problem that certificates cannot be revoked, and what mitigates it?
4. Which method is best for human users in a company?
5. On EKS, what replaces the `aws-auth` ConfigMap?
6. Why bind roles to groups instead of users?

<details>
<summary>Answers</summary>

1. From the authentication layer (certificates, OIDC, cloud IAM, tokens). Kubernetes does not store them
2. `CN` is the username and `O` is the group (several `O` values give several groups)
3. A leaked certificate stays valid until it expires. Use short expiry and prefer OIDC or IAM for people
4. OIDC (or cloud IAM on managed clusters)
5. EKS access entries (the Cluster Access Management API)
6. Access follows team membership: change the group in the identity provider and no bindings need editing

</details>

---

[← Bindings](rolebindings-and-clusterrolebindings.md) | [Module README](../README.md) | [Testing and Debugging →](testing-and-debugging.md)
