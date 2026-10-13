# Module 09: Security

> Secure the cluster: ServiceAccounts, RBAC, security contexts, Pod Security and network security.

**Notes ready:** 9/9 notes (5 topics, with RBAC split into 6 pages)

## Topic directory

| Topic | File | Question it answers | Status |
|---|---|---|---|
| **Service Accounts** | [`service-accounts.md`](service-accounts.md) | What identity does a Pod have? | 📘 |
| **RBAC** | [`RBAC/`](RBAC/README.md) | Who may do what in the cluster? | 📘 |
| **Security Context** | [`security-context.md`](security-context.md) | What may a container do on the node? | 📘 |
| **Pod Security** | [`pod-security.md`](pod-security.md) | How do I force every Pod to meet a standard? | 📘 |
| **Network Security** | [`network-security.md`](network-security.md) | Who can talk to whom, and what can leave? | 📘 |

### The RBAC pages

| Page | File | You learn |
|---|---|---|
| Overview | [`RBAC/README.md`](RBAC/README.md) | The mental model, the four objects, built-in roles |
| Roles | [`RBAC/roles-and-clusterroles.md`](RBAC/roles-and-clusterroles.md) | Rule anatomy, verbs, subresources, which kubectl command needs what |
| Bindings | [`RBAC/rolebindings-and-clusterrolebindings.md`](RBAC/rolebindings-and-clusterrolebindings.md) | Subjects, `roleRef`, the four combinations |
| Users and groups | [`RBAC/users-and-groups.md`](RBAC/users-and-groups.md) | Certificates, OIDC, EKS access entries |
| Testing | [`RBAC/testing-and-debugging.md`](RBAC/testing-and-debugging.md) | `kubectl auth can-i`, impersonation, a fix-the-bug drill |
| Best practices | [`RBAC/best-practices.md`](RBAC/best-practices.md) | Least privilege, dangerous permissions, CI service accounts |

YAML for the labs: [`labs/08-rbac/manifests/`](../labs/08-rbac/manifests/). Combined lab: [`labs/08-rbac`](../labs/08-rbac/).

## The big idea: layers of defense

No single control is enough. Each one stops a different step of an attack:

```
Request enters the cluster
  1. Authentication      who are you?                          certificates, OIDC, IAM, tokens
  2. Authorization       what may you do?                      RBAC
  3. Admission           is the object acceptable?             Pod Security, quotas, policies
  4. Runtime             what may the container do?            securityContext (user, capabilities, filesystem)
  5. Network             who may it talk to, and where to?     NetworkPolicy, TLS, egress control
  6. Identity of the Pod what may it call?                     ServiceAccount, IRSA / Pod Identity
```

If one layer fails, the next one limits the damage. This is **defense in depth**.

## Which control stops what?

| Attack or mistake | Stopped by |
|---|---|
| A stranger calls the API | Authentication |
| A developer deletes production | RBAC (no write access in prod) |
| Someone creates a privileged Pod | **Pod Security** (and RBAC on who can create Pods) |
| A compromised container gains root or loads tools | **Security context** (non-root, no capabilities, read-only filesystem) |
| A compromised Pod calls the Kubernetes API | No mounted token, **minimal ServiceAccount RBAC** |
| A compromised Pod scans other Pods or reaches the database | **NetworkPolicy** default deny |
| A Pod steals the node's cloud credentials | IRSA or Pod Identity, metadata hardening, egress block |
| Data leaves to the internet | Egress policy |
| Traffic between services is sniffed | mTLS or CNI encryption |
| A pipeline's leaked credential wipes the cluster | Namespace-scoped, short-lived CI ServiceAccount |

## The same question, three layers

"Can this Pod do something dangerous?" is answered at three different levels, which people often confuse:

| | **RBAC** | **Security context / Pod Security** | **NetworkPolicy** |
|---|---|---|---|
| Controls | Access to the **Kubernetes API** | What a container can do on the **node** | **Network** connections |
| Applies to | Users, groups, ServiceAccounts | Pod spec | Pods |
| Example | "Can `app-sa` list Secrets?" | "Can this container run as root?" | "Can this Pod reach the database?" |

## Study order

| # | Topic | Why now |
|---|---|---|
| 1 | [Service Accounts](service-accounts.md) | The identity everything else attaches to |
| 2 | [RBAC overview](RBAC/README.md) → [Roles](RBAC/roles-and-clusterroles.md) → [Bindings](RBAC/rolebindings-and-clusterrolebindings.md) | The core of authorization |
| 3 | [Users and groups](RBAC/users-and-groups.md) | Where human identities come from |
| 4 | [Testing and debugging](RBAC/testing-and-debugging.md) | Verify every grant |
| 5 | [RBAC best practices](RBAC/best-practices.md) | Least privilege and escalation traps |
| 6 | [Security Context](security-context.md) | Harden the container itself |
| 7 | [Pod Security](pod-security.md) | Enforce it for everyone |
| 8 | [Network Security](network-security.md) | Limit the blast radius |

## Starter hardening checklist

**Identity and access**

- [ ] Nobody uses `cluster-admin` for daily work
- [ ] People get access through **groups**, and humans authenticate with OIDC or cloud IAM
- [ ] Every workload has its **own** ServiceAccount, and `default` has no permissions
- [ ] `automountServiceAccountToken: false` wherever the API is not needed
- [ ] CI/CD identities are **namespace-scoped and short-lived**

**Workloads**

- [ ] Containers run as **non-root**, drop **ALL** capabilities, block privilege escalation
- [ ] **Read-only root filesystem** with `emptyDir` where the app must write
- [ ] Application namespaces enforce **Baseline** or **Restricted** Pod Security
- [ ] Images are pinned (no `:latest`) and come from a trusted registry

**Network and data**

- [ ] **Default-deny** NetworkPolicy in every application namespace, plus explicit allows (including DNS)
- [ ] **Egress** is restricted, and the cloud metadata service is not reachable from Pods
- [ ] Secrets are encrypted at rest, and mTLS or encryption protects traffic between services
- [ ] The API server endpoint is private or restricted by source address

**Operations**

- [ ] RBAC and policies live in **Git** and go through review
- [ ] **Audit logging** is on
- [ ] There is a tested **break-glass** administrator path

## Labs

- [labs/08-rbac](../labs/08-rbac/): ServiceAccount calls to the API, roles and bindings with impersonation, a certificate user, a broken-RBAC drill, the Pod-creation escalation, security contexts, Pod Security, network audit, and a combined lockdown of one namespace
- Each note has its own lab. The NetworkPolicy parts need a CNI that enforces policies: `minikube start -p np --cni=calico`.

## Module quiz

1. In what order do authentication, authorization and admission happen?
2. Why can RBAC never deny something?
3. What is the difference between a Role and a RoleBinding?
4. What does a RoleBinding that references a ClusterRole do?
5. Where do Kubernetes usernames and groups come from?
6. Which ServiceAccount does a Pod use if you do not set one, and why not bind roles to it?
7. How are modern ServiceAccount tokens different from the old Secret-based ones?
8. Why is the ability to create Pods in a namespace effectively access to its Secrets?
9. Name three permissions that allow privilege escalation.
10. What does `runAsNonRoot: true` do when the image runs as root?
11. What does `allowPrivilegeEscalation: false` do?
12. What are the three Pod Security profiles and the three PSA modes?
13. How do you preview what enforcing a Pod Security level would break?
14. Why is a NetworkPolicy useless without a suitable CNI?
15. Why must egress be controlled, and what does `169.254.169.254` have to do with it?
16. Which command shows everything a user may do in a namespace?
17. What is the difference between a 401 and a 403?
18. How should a CI pipeline get access to a cluster?

<details>
<summary>Answers</summary>

1. Authentication, then authorization (RBAC), then admission control
2. Rules only grant. Permissions are the union of all rules, and anything unlisted is denied by default
3. A Role defines permissions. A RoleBinding gives that role to subjects
4. It grants the ClusterRole's rules only inside the RoleBinding's namespace
5. From the authentication layer (certificates, OIDC, cloud IAM, tokens). Kubernetes does not store users or groups
6. The namespace's `default` ServiceAccount. Roles bound to it would apply to every Pod that did not choose an account
7. They are bound to the Pod, expire and are refreshed, and are audience-bound. The old Secrets never expired
8. A new Pod can mount any Secret in the namespace (or run as any ServiceAccount) and reveal the value
9. For example `create pods`, `pods/exec`, `serviceaccounts/token`, `impersonate`, `escalate`, `bind`, editing RBAC objects, `list` on secrets
10. The container fails to start with `CreateContainerConfigError`
11. It sets `no_new_privs`, so a process cannot gain more privileges than it started with (no setuid escalation)
12. Profiles: Privileged, Baseline, Restricted. Modes: enforce, warn, audit
13. `kubectl label --dry-run=server --overwrite ns <ns> pod-security.kubernetes.io/enforce=<level>`
14. The CNI plugin enforces the rules. Without enforcement the API accepts policies that silently do nothing
15. Compromised Pods use egress to exfiltrate data and reach internal systems. The metadata address hands out the node's cloud credentials
16. `kubectl auth can-i --list -n <ns> --as=<user>` (add `--as-group`)
17. 401: authentication failed. 403: authenticated but not authorized
18. With a namespace-scoped ServiceAccount (or federated identity) and short-lived credentials, never a stored cluster-admin kubeconfig

</details>

## Self-test before moving on

1. Grant a user read access to Pods in one namespace only, and prove it with `kubectl auth can-i --as`.
2. Fix the four broken RBAC setups without looking at the solutions.
3. Show how someone who can only create Pods reads a Secret, then block it with Pod Security and namespace design.
4. Run a Pod that passes the Restricted profile.
5. Lock one namespace down end to end with `lockdown.yaml` and attack it from the `intruder` Pod.
6. Take the module quiz without looking at the answers.

## How to study this module

1. Read each note in order.
2. Type the YAML yourself, do not copy-paste.
3. Run the lab, then break it on purpose.
4. When you have studied and practiced a note, change its `Status` line to `✅ Studied`.

Next: [Module 10: Helm](../10-helm/)
