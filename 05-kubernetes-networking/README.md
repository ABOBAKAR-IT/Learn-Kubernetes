# Module 05: Kubernetes Networking

> Connect and expose workloads: Pod networking, Services, DNS, Ingress and NetworkPolicy.

**Notes ready:** 7/7 topics

## Topic directory

| Topic | File | Question it answers | Status |
|---|---|---|---|
| **Pod Networking** | [`pod-networking.md`](pod-networking.md) | How does a Pod get an IP and reach other Pods? | 📘 |
| **Services (overview)** | [`services/README.md`](services/README.md) | Why do Services exist and how do they work? | 📘 |
| **ClusterIP** | [`services/clusterip.md`](services/clusterip.md) | How do Pods call each other reliably? | 📘 |
| **NodePort** | [`services/nodeport.md`](services/nodeport.md) | How do I reach a Service from outside a local cluster? | 📘 |
| **LoadBalancer** | [`services/loadbalancer.md`](services/loadbalancer.md) | How do I get a public address from the cloud? | 📘 |
| **DNS** | [`dns.md`](dns.md) | How do names like `echo` resolve? | 📘 |
| **Ingress** | [`ingress/README.md`](ingress/README.md) | How do I route many web apps through one entry point? | 📘 |
| **Network Policies** | [`network-policies/README.md`](network-policies/README.md) | Who is allowed to talk to whom? | 📘 |

YAML for the labs: [`labs/03-service/manifests/`](../labs/03-service/manifests/) (Pods, Services, DNS) and [`labs/05-ingress/manifests/`](../labs/05-ingress/manifests/) (Ingress, NetworkPolicy).

## The big idea: five questions, five answers

```
1. How do Pods reach each other?        flat Pod network, one IP per Pod      → CNI plugin
2. How do I reach "the app", not a Pod? stable virtual IP + load balancing    → Service (kube-proxy)
3. How do I find it by name?            service names in DNS                   → CoreDNS
4. How do outsiders get in?             NodePort / LoadBalancer / Ingress      → cloud LB, controller
5. Who may talk to whom?                label-based firewall                   → NetworkPolicy
```

## The full path of a request

```
Internet
   │
   ▼
Cloud load balancer ── (LoadBalancer Service, or in front of the Ingress controller)
   │
   ▼
Ingress controller ── picks the Service by host and path
   │
   ▼
Service (ClusterIP) ── kube-proxy rules choose a Ready Pod
   │
   ▼
Pod ── reached over the flat Pod network (CNI), allowed by NetworkPolicy
```

DNS (CoreDNS) is used at several steps: users resolve your public name, and Pods resolve Service names.

## Which way in? (exposing an app)

| I want to... | Use |
|---|---|
| Let Pods call a backend | **ClusterIP** |
| Test from my laptop on a local cluster | **NodePort** or `kubectl port-forward` |
| Expose one TCP/UDP service publicly | **LoadBalancer** |
| Serve many web apps with host/path rules and TLS | **Ingress** (or Gateway API) |
| Point an in-cluster name at an outside host | **ExternalName** |
| Give StatefulSet Pods their own DNS names | **Headless** Service |

## Comparison

| | ClusterIP | NodePort | LoadBalancer | Ingress |
|---|---|---|---|---|
| Reachable from | Cluster | Node IPs | Internet or VPC | Internet or VPC |
| Layer | 4 | 4 | 4 | **7 (HTTP/S)** |
| Needs a cloud | No | No | **Yes** | A controller (usually behind a LB) |
| One per... | Service | Service | **Service (billable)** | Many Services |
| TLS and host/path routing | No | No | No | **Yes** |

## Recommended study order

| # | Topic | Why now |
|---|---|---|
| 1 | [Pod Networking](pod-networking.md) | The foundation everything sits on |
| 2 | [Services overview](services/README.md) | The core idea |
| 3 | [ClusterIP](services/clusterip.md) | The default type |
| 4 | [NodePort](services/nodeport.md) | Builds on ClusterIP |
| 5 | [LoadBalancer](services/loadbalancer.md) | Builds on NodePort |
| 6 | [DNS](dns.md) | How names map to Services |
| 7 | [Ingress](ingress/README.md) | Uses Services and DNS |
| 8 | [Network Policies](network-policies/README.md) | Restrict the paths you have opened |

## Labs

- [labs/03-service](../labs/03-service/): the welcome-app lab (Deployment + NodePort + LoadBalancer on minikube)
- [labs/05-ingress](../labs/05-ingress/): path, host and TLS routing through an Ingress controller
- Each note has its own lab. The NetworkPolicy lab needs `minikube start --cni=calico`.

## Module quiz

1. What are the rules of the Kubernetes network model?
2. Who implements Pod networking, Kubernetes itself or a CNI plugin?
3. Why do Services exist if Pods already have IPs?
4. How do ClusterIP, NodePort and LoadBalancer relate to each other?
5. What is the difference between `port`, `targetPort` and `nodePort`?
6. A Service has no endpoints. What are the first two things you check?
7. On minikube, `EXTERNAL-IP` stays `<pending>`. Why, and what fixes it?
8. What is the full DNS name of Service `api` in namespace `shop`?
9. What does `ndots:5` do to a lookup of `api.example.com`?
10. What is the difference between an Ingress and an Ingress controller?
11. Why is one Ingress cheaper than ten LoadBalancer Services?
12. With no NetworkPolicy in a namespace, what traffic is allowed?
13. After adding a default-deny egress policy, DNS stops working. Why?
14. What is the difference between one `from` item containing two selectors and two `from` items?
15. A Pod reaches a Service by its ClusterIP but not by name. Where do you look?
16. You need to expose a non-HTTP TCP service publicly. Which type do you use?

<details>
<summary>Answers</summary>

1. Every Pod has its own IP. Pods reach all Pods without NAT. Node agents reach all Pods on their node. Containers in one Pod share a network namespace
2. A CNI plugin (Calico, Cilium, VPC CNI...). Kubernetes defines the model only
3. Pod IPs change. A Service gives a stable virtual IP, a DNS name and load balancing across Ready Pods
4. LoadBalancer includes a NodePort, which includes a ClusterIP
5. `port`: the Service's port. `targetPort`: the Pod's port. `nodePort`: the port opened on every node
6. The Service selector against the Pod labels, and whether the Pods are Ready
7. There is no cloud load balancer. Run `minikube tunnel`
8. `api.shop.svc.cluster.local`
9. Because it has fewer than 5 dots, several search suffixes are tried (and fail) before the real name is looked up
10. The Ingress is the routing rules object. The controller is the proxy software that enforces them
11. One shared load balancer and controller serve many hosts and paths
12. All traffic. Pods are not isolated
13. Egress to CoreDNS on port 53 is blocked. Add an allow rule for it
14. One item with two selectors is AND. Two items is OR
15. CoreDNS health and logs, `/etc/resolv.conf` in the Pod, and any policy blocking port 53
16. LoadBalancer (Ingress is HTTP/HTTPS only)

</details>

## Self-test before moving on

1. Draw the path of a request from the internet to a Pod and label each component.
2. Break a Service on purpose (wrong selector), then find the cause using only `kubectl get endpointslices` and `--show-labels`.
3. Make DNS fail and show that access by IP still works.
4. Apply a default-deny policy and then open exactly one path.
5. Take the module quiz without looking at the answers.

## How to study this module

1. Read each note in order.
2. Type the YAML yourself, do not copy-paste.
3. Run the lab, then break it on purpose.
4. When you have studied and practiced a note, change its `Status` line to `✅ Studied`.

Next: [Module 08: Health and Scaling](../08-health-and-scaling/) in the recommended order, or [Module 07: Scheduling](../07-scheduling/)