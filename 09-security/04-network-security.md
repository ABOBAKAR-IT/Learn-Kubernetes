# Network Security

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/08-rbac/manifests/`](../labs/08-rbac/manifests/) · Lab: [`labs/08-rbac`](../labs/08-rbac/)

[← Pod Security](pod-security.md) | [README](README.md)

---

## Idea

RBAC controls the **API**. Security contexts and Pod Security control **what runs**. **Network security** controls **who can talk to whom**, and what can leave the cluster. The mechanics of NetworkPolicy are in [Module 05](../05-kubernetes-networking/network-policies/README.md). This note is about the **security design** around it.

```
Default Kubernetes network:  every Pod can reach every Pod, the internet, and often cloud internals

Secure design:  deny by default  →  allow only the flows you can explain
```

## The layers

| Layer | Question | Tools |
|---|---|---|
| **Cluster edge** | Who can reach the API and the apps from outside? | API endpoint access rules, Ingress, load balancer and security groups |
| **Between workloads** | Which Pods may talk to which? | **NetworkPolicy**, service mesh policies |
| **Leaving the cluster** | What may Pods reach on the internet and your VPC? | Egress NetworkPolicy, NAT and firewall |
| **Cloud internals** | Can a Pod steal the node's cloud credentials? | Metadata service hardening, IRSA, Pod Identity |
| **On the wire** | Can traffic be read or changed? | TLS, mTLS, encrypted CNI |
| **The nodes** | Are node ports and the kubelet exposed? | Security groups, kubelet authentication |

## 1. Default deny, then allow

Every application namespace should start closed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

Then add **explicit allows**: ingress controller → web, web → api, api → database, everything → DNS. For egress, use default deny plus allows for DNS, your dependencies and the internet where required. The patterns and a hands-on lab are in [Network Policies](../05-kubernetes-networking/network-policies/README.md).

Two reminders:

- Policies need a **CNI that enforces them** (Calico, Cilium, Antrea, or the AWS VPC CNI with network policy support enabled). An unenforced policy is silent.
- Policies cover **layer 3 and 4** (IPs, ports). They do not inspect HTTP or verify identity.

## 2. Control egress

Compromised Pods usually try to **phone home** or **reach internal systems**. Limit what they can reach:

```yaml
egress:
  # DNS
  - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
        podSelector:
          matchLabels:
            k8s-app: kube-dns
    ports:
      - { protocol: UDP, port: 53 }
      - { protocol: TCP, port: 53 }
  # the public internet over HTTPS only, never private ranges or the metadata address
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0
          except: [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 169.254.169.254/32]
    ports:
      - { protocol: TCP, port: 443 }
```

Plain NetworkPolicy knows **IP ranges, not hostnames**. To allow "only `api.partner.com`", use a CNI feature (for example Cilium or Calico DNS or FQDN policies) or an egress proxy. Sending egress through a NAT gateway or proxy also gives you **one fixed source address** that partners can allow-list.

## 3. Protect the cloud metadata service (AWS)

Every EC2 node can reach the **instance metadata service** at `169.254.169.254`, which hands out the **node's IAM role credentials**. A Pod that can reach it may use **the node's permissions**, often far more than the app needs.

Defenses, strongest together:

| Defense | How |
|---|---|
| **Use IRSA or EKS Pod Identity** | Give each workload its own narrow IAM role (see [ServiceAccounts](service-accounts.md)). The node role stays minimal |
| **Require IMDSv2 with hop limit 1** | A token-based metadata service whose responses cannot cross the extra network hop into a Pod. Set it in the node launch template (`http_tokens = required`, `http_put_response_hop_limit = 1`) |
| **Block the address with egress policy** | Include `169.254.169.254/32` in the `except` list above |
| **Keep node roles small** | Only what nodes need to join and pull images |

## 4. Expose only what you must

Review how things reach the outside:

| Exposure | Check |
|---|---|
| `type: NodePort` | Opens a port on **every node**. Is the security group open to it? |
| `type: LoadBalancer` | Public by default. Use the internal scheme for private services and restrict source ranges |
| **Ingress** | Terminate **TLS**, redirect HTTP to HTTPS, limit who can reach it |
| `hostNetwork`, `hostPort` | Bypass Pod networking. Pod Security forbids them under Baseline |
| `kubectl port-forward`, `kubectl proxy` | Debugging tools, protected by RBAC (`pods/portforward`) |

## 5. Encryption in transit

| Where | How |
|---|---|
| Client → Ingress | TLS certificate (cert-manager or a cloud certificate service) |
| API server, etcd, kubelet | TLS by default, handled by the cluster |
| **Pod → Pod** | **Not encrypted by default.** Add **mTLS** with a service mesh (Istio, Linkerd), or **transparent encryption** in the CNI (for example WireGuard in Cilium or Calico) |
| To databases and other services | TLS in the connection settings of your app |

mTLS also gives **identity**: services can check **who** is calling, not just from which IP.

## 6. Secure the API server and nodes

| Area | Practice |
|---|---|
| **API server endpoint** | Prefer a **private** endpoint. If it must be public, **restrict source CIDRs** (EKS supports this) |
| **Anonymous access** | Disabled for the API server and kubelet |
| **kubelet** | Authentication and authorization enabled (webhook), read-only port off |
| **etcd** | Reachable only from the API servers, with TLS and encryption at rest for Secrets |
| **Node security groups** | Only required traffic. Do not expose node ports to the world |
| **Security groups for Pods** (EKS, VPC CNI) | Attach AWS security groups to specific Pods |

## Threats and what stops them

| Threat | First defense | Second defense |
|---|---|---|
| A compromised web Pod scans other Pods | Default-deny **NetworkPolicy** | Namespaces, service mesh policy |
| It calls the Kubernetes API with its token | **No token mounted**, minimal RBAC | Audit logging |
| It steals the node's cloud credentials | **IRSA or Pod Identity**, IMDSv2 hop limit 1 | Egress block on `169.254.169.254` |
| It exfiltrates data to the internet | **Egress policy** | Egress proxy and monitoring |
| Traffic is sniffed inside the cluster | **mTLS or CNI encryption** | Per-service TLS |
| A node port is open to the world | Security groups | Review of Service types |
| Someone mounts the host network | **Pod Security** (Baseline) | Admission policies |

## Lab

**1. Audit your cluster for common exposure**

```bash
cd labs/08-rbac
./security-audit.sh
```

It prints, read-only: externally reachable Services, Pods on the host network or PID namespace, privileged containers, namespaces **without** NetworkPolicy, namespaces without Pod Security enforcement, `cluster-admin` bindings, and ServiceAccount token settings. Discuss each line: is it expected?

**2. Lock a namespace down to the internet only** (needs a CNI that enforces policies: `minikube start -p np --cni=calico`)

```bash
cd manifests
kubectl apply -f secure-apps.yaml
kubectl wait --for=condition=Ready pod/client -n secure-apps --timeout=90s

# something private to try to reach
kubectl run target --image=nginx:1.27
TARGET=$(kubectl get pod target -o jsonpath='{.status.podIP}'); echo $TARGET

# DNS works, and the public internet over HTTPS works
kubectl exec -n secure-apps client -- nslookup example.com | tail -3
kubectl exec -n secure-apps client -- curl -sS -m 5 -o /dev/null -w '%{http_code}\n' https://example.com

# a private Pod address is blocked (timeout)
kubectl exec -n secure-apps client -- curl -sS -m 5 http://$TARGET
```

The last call should time out: private ranges are excluded. Without an enforcing CNI, it will succeed, which shows why you must verify enforcement.

**2b. A default-deny baseline for both directions** (same CNI requirement)

```bash
kubectl apply -f netpol-baseline.yaml                         # namespace "secure": deny all ingress and egress, then allow DNS
kubectl run probe -n secure --image=busybox:1.36 -- sleep 3600
kubectl wait --for=condition=Ready pod/probe -n secure --timeout=60s
kubectl exec -n secure probe -- nslookup kubernetes.default | tail -3        # DNS works
kubectl exec -n secure probe -- wget -qO- -T 3 http://$TARGET                # blocked: no egress allowed except DNS
```

Everything except DNS is closed. Each new flow now needs an explicit allow, which is the habit you want.

**3. Full lockdown: roles, Pod Security, NetworkPolicy together**

The combined lab in [`labs/08-rbac`](../labs/08-rbac/) (`lockdown.yaml`) builds one namespace with all the controls from this module and tests them.

**Cleanup**

```bash
kubectl delete -f secure-apps.yaml -f netpol-baseline.yaml
kubectl delete pod target
```

## Gotchas

- **Default allow** is the Kubernetes default. A cluster with no NetworkPolicy is a flat, open network.
- **Egress is forgotten.** Teams lock down ingress and leave outbound traffic open, which is how stolen data leaves.
- **IP-based egress rules cannot allow by hostname.** Plan for FQDN support or a proxy.
- **Default-deny egress breaks DNS** unless you allow port 53 to CoreDNS.
- **The metadata service is a credential store**, not just an information endpoint.
- **Encryption is off between Pods by default.** "Inside the cluster" is not "safe".
- A **service mesh adds identity and encryption** but also complexity and resource cost. Adopt it deliberately.
- NetworkPolicy does not protect against a compromised **node** or against anyone who can edit the policies, so guard RBAC for NetworkPolicy objects too.

## Check yourself

1. What is the default network posture of a Kubernetes cluster?
2. Why is controlling **egress** as important as ingress?
3. How can a Pod abuse the cloud metadata service, and how do you stop it on AWS?
4. Which traffic is **not** encrypted by default, and how do you encrypt it?
5. What can a plain NetworkPolicy not do about hostnames?
6. Which three commands or checks find exposed Services or privileged Pods in a cluster?

<details>
<summary>Answers</summary>

1. Default allow: every Pod can reach every other Pod and usually the internet
2. Compromised Pods use egress to exfiltrate data, call home and reach internal systems or credentials
3. It can fetch the node's IAM credentials from `169.254.169.254`. Use IRSA or Pod Identity, require IMDSv2 with hop limit 1, block the address with egress policy and keep node roles small
4. Pod-to-Pod traffic. Use mTLS from a service mesh or transparent encryption in the CNI (such as WireGuard)
5. It filters IP ranges and ports only, so allowing a hostname needs CNI FQDN features or an egress proxy
6. For example `kubectl get svc -A` (NodePort/LoadBalancer types), querying Pods for `hostNetwork` and `privileged`, and `kubectl get netpol -A`, all combined in `security-audit.sh`

</details>

---

[← Pod Security](pod-security.md) | [README](README.md)
