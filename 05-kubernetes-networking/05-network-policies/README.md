# Network Policies

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/05-ingress/manifests/`](../../labs/05-ingress/manifests/)

[← Ingress](../ingress/README.md) | [Module README](../README.md)

---

## Idea

By default **every Pod can talk to every other Pod**, in every namespace. A **NetworkPolicy** is a firewall rule set for Pods (layer 3/4: IPs, ports, protocols), written with **labels**.

```
no policy:         every Pod ◄──► every Pod            (open)
with a policy:     only the traffic the policies allow  (everything else dropped)
```

## Requirement: your CNI must enforce policies

The API accepts a NetworkPolicy even if nothing enforces it, and then **it silently does nothing**.

| CNI | Enforces NetworkPolicy? |
|---|---|
| Calico, Cilium, Antrea | Yes |
| AWS VPC CNI | Yes in recent versions, when network policy support is enabled |
| Flannel (alone) | **No** |
| Basic local defaults (minikube, kind) | Often **no** |

For the lab, start minikube with Calico: `minikube start -p np --cni=calico`.

## The rules of the game

| Rule | Meaning |
|---|---|
| **No policy selects a Pod** | That Pod accepts and sends everything |
| **A policy selects a Pod** | The Pod becomes **isolated** for the directions that policy lists (`Ingress`, `Egress`) |
| Policies are **additive allow-lists** | There are no "deny" rules. Allowed traffic is the **union** of all policies selecting the Pod |
| **Replies are allowed** | Connections are stateful. Allowing a request allows its response |
| Both ends matter | For A → B: A's **egress** (if restricted) **and** B's **ingress** (if restricted) must allow it |
| **Namespaced** | A policy only selects Pods in its own namespace |

## Anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: shop
spec:
  podSelector:               # WHICH Pods this policy protects (the targets)
    matchLabels:
      role: backend
  policyTypes: ["Ingress"]   # which directions it controls (always list them explicitly)
  ingress:                   # who may connect IN
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 80
```

Read it as: *"Pods labeled role=backend accept TCP port 80 from Pods labeled role=frontend, and nothing else."*

| Field | Meaning |
|---|---|
| `podSelector` | The Pods the policy applies to. `{}` means **all Pods in the namespace** |
| `policyTypes` | `Ingress`, `Egress`, or both |
| `ingress[].from` / `egress[].to` | The allowed peers |
| `ports` | Allowed protocols and ports (`endPort` allows a range) |

### Peers: three ways to say who

| Peer | Selects |
|---|---|
| `podSelector` | Pods in the **same** namespace (or with a `namespaceSelector`, in those namespaces) |
| `namespaceSelector` | All Pods in matching namespaces |
| `ipBlock` | A CIDR range, with optional `except` (used for outside addresses) |

```yaml
- ipBlock:
    cidr: 10.0.0.0/16
    except:
      - 10.0.5.0/24
```

Every namespace carries the label `kubernetes.io/metadata.name: <name>`, which you can use in a `namespaceSelector`.

### The AND vs OR trap

**One list item with two selectors = AND** (Pods labeled `role=frontend` **in** namespaces labeled `team=a`):

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels: { team: a }
        podSelector:
          matchLabels: { role: frontend }
```

**Two list items = OR** (any Pod in a `team=a` namespace, **or** any `role=frontend` Pod in this namespace):

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels: { team: a }
      - podSelector:
          matchLabels: { role: frontend }
```

One missing indentation level changes the meaning. Review these carefully.

## The standard patterns

**1. Default deny all ingress in a namespace** (start here)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: shop
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

**2. Default deny all egress, but allow DNS** (forgetting DNS is the classic mistake)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress-allow-dns
  namespace: shop
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**3. Allow only specific callers** (the example in the Anatomy section)

**4. Allow from another namespace**

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
```

**5. Allow egress to an external range**

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 203.0.113.0/24
    ports:
      - protocol: TCP
        port: 443
```

## Lab

Needs a CNI that enforces policies: `minikube start -p np --cni=calico` (then `kubectl config use-context np`).

```bash
cd labs/05-ingress/manifests
kubectl apply -f np-lab.yaml                          # namespace shop: frontend, intruder, backend (+Service)
kubectl -n shop wait --for=condition=Ready pod --all --timeout=180s
```

**1. Baseline: everything is open**

```bash
kubectl -n shop exec frontend -- wget -qO- -T 3 http://backend | head -4     # works
kubectl -n shop exec intruder -- wget -qO- -T 3 http://backend | head -4     # also works
```

**2. Default deny ingress: backend is now closed**

```bash
kubectl apply -f default-deny-ingress.yaml
kubectl -n shop exec frontend -- wget -qO- -T 3 http://backend               # times out
```

**3. Allow only the frontend**

```bash
kubectl apply -f allow-frontend-to-backend.yaml
kubectl -n shop exec frontend -- wget -qO- -T 3 http://backend | head -4     # works
kubectl -n shop exec intruder -- wget -qO- -T 3 http://backend               # still blocked
```

**4. Lock down egress too**

```bash
kubectl apply -f default-deny-egress-allow-dns.yaml
kubectl -n shop exec frontend -- nslookup backend                            # DNS still works
kubectl -n shop exec frontend -- wget -qO- -T 3 http://backend               # blocked: egress denied
kubectl apply -f allow-frontend-egress-to-backend.yaml
kubectl -n shop exec frontend -- wget -qO- -T 3 http://backend | head -4     # works again
```

**5. Inspect**

```bash
kubectl -n shop get networkpolicy
kubectl -n shop describe networkpolicy allow-frontend-to-backend
```

**Cleanup**

```bash
kubectl delete namespace shop
```

## Debugging

| Symptom | Likely cause |
|---|---|
| Policy has **no effect** at all | The CNI does not enforce NetworkPolicy |
| DNS stopped working after an egress policy | Port 53 to CoreDNS not allowed |
| A allowed to B in one policy but still blocked | The other side (A's egress or B's ingress) has a restricting policy |
| Wrong Pods affected | `podSelector` labels do not match, or the policy is in another namespace |
| Cross-namespace traffic blocked | Missing `namespaceSelector` for the source namespace |

```bash
kubectl get pods --show-labels -n shop
kubectl get netpol -A
```

Quick test method: run a throwaway Pod with the labels of the client you are testing and use `wget -T 3` or `nc -zv -w 3 <host> <port>`.

## Best practice

1. In each namespace, start with **default deny** for ingress (and egress if you can manage it).
2. **Allow explicitly**, for each flow: frontend → backend, backend → database, everything → DNS.
3. Use meaningful, stable **labels** (`app.kubernetes.io/name`, `role`).
4. Test policies in a non-production namespace and with a CNI that logs drops.
5. Pair with RBAC and Pod Security. See [network security](../../09-security/network-security.md).

## Gotchas

- **Silent failure:** an unenforced policy looks fine and does nothing.
- Policies are **allow-only**: you cannot write "deny this one Pod".
- `podSelector: {}` is **everything in the namespace**, not "nothing".
- An empty `ingress: []` with `policyTypes: ["Ingress"]` blocks all ingress. A rule with `from: []` or an empty rule can allow **all**, so read carefully.
- Node and kubelet probe traffic behaviour depends on the CNI. Test that readiness probes still work after applying default-deny.
- NetworkPolicy does not do HTTP-level rules or TLS. A service mesh or L7 policies do that.

## Check yourself

1. What happens to traffic when **no** NetworkPolicy selects a Pod?
2. What does a policy with `podSelector: {}` and no rules select and allow?
3. Are NetworkPolicies allow rules, deny rules, or both?
4. What is the difference between one `from` item with two selectors and two `from` items?
5. After adding default-deny egress, name resolution fails. Why, and how do you fix it?
6. What must be true of the cluster for policies to work?

<details>
<summary>Answers</summary>

1. It is not isolated: all traffic is allowed
2. It selects every Pod in the namespace and, with a policy type but no rules, allows nothing in that direction (default deny)
3. Only allow rules. Allowed traffic is the union of all policies selecting the Pod
4. One item with two selectors is AND (both must match). Two items is OR (either matches)
5. Egress to CoreDNS on port 53 (UDP and TCP) is blocked. Add an egress rule allowing it to the kube-dns Pods in `kube-system`
6. The CNI plugin must enforce NetworkPolicy (Calico, Cilium, Antrea, VPC CNI with support enabled...)

</details>

---

[← Ingress](../ingress/README.md) | [Module README](../README.md)