# Pod Networking

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/03-service/manifests/`](../labs/03-service/manifests/)

[README](README.md) | [Services →](services/README.md)

---

## Idea

Every Pod gets **its own IP address**, and any Pod can reach any other Pod directly by that IP, on any node, **without NAT**. This flat network is the foundation. Services, DNS and Ingress are all built on top of it.

## The Kubernetes network model

| Rule | Meaning |
|---|---|
| 1. Every Pod has its **own IP** | Not shared with other Pods |
| 2. Pods reach all other Pods **without NAT** | The sender's IP is what the receiver sees |
| 3. Agents on a node (kubelet, system daemons) reach all Pods on that node | Used for probes and node services |
| 4. Containers **inside one Pod** share one network namespace | They talk over `localhost` and cannot reuse the same port |

Kubernetes defines these rules but does not implement them. A **CNI plugin** does.

## How a Pod gets its network

```
1. kubelet asks the runtime to create the Pod sandbox (the "pause" container)
2. The runtime calls the CNI plugin:  "set up networking for this sandbox"
3. The plugin: allocates an IP from the node's Pod range
               creates a virtual cable (veth pair) between the Pod and the node
               adds routes / tunnels so other nodes can reach that IP
4. All containers of the Pod join the sandbox's network namespace
```

```
              Node 1                                    Node 2
   ┌──────────────────────────┐              ┌──────────────────────────┐
   │ Pod A 10.244.1.4         │              │ Pod C 10.244.2.7         │
   │   eth0 ──veth── bridge/  │ ─────────────┼─► routes or overlay      │
   │ Pod B 10.244.1.5         │   node network│    (VXLAN, BGP, VPC)     │
   └──────────────────────────┘              └──────────────────────────┘
   A → B: same node, through the bridge      A → C: crosses the node network
```

## CNI plugins

The **Container Network Interface** is the standard way the runtime asks a plugin to wire up a Pod.

| Plugin | Approach | Notable |
|---|---|---|
| **Calico** | Routing (BGP) or overlay, optional eBPF | Strong **NetworkPolicy** support |
| **Cilium** | eBPF | NetworkPolicy plus observability, can replace kube-proxy |
| **Flannel** | Simple VXLAN overlay | Easy, **no NetworkPolicy** |
| **AWS VPC CNI** | Pods get real **VPC IP addresses** (from ENIs) | EKS default, Pod IPs routable inside the VPC |
| **kindnet / bridge** | Basic local setups | Default in kind and minikube, limited features |

**NetworkPolicy is only enforced if your CNI supports it.** See [network-policies](network-policies/README.md).

## Three address ranges (do not mix them up)

| Range | Used for | Where it comes from |
|---|---|---|
| **Node IPs** | The machines | Your network or cloud |
| **Pod CIDR** | Pod IPs | Split per node by the cluster and CNI |
| **Service CIDR** | Service virtual IPs (ClusterIP) | A separate range reserved for Services |

Service IPs are **virtual**: no interface owns them. Rules written by kube-proxy translate them to Pod IPs.

## Lab

```bash
# 1. Pods and their IPs
kubectl create deployment web --image=nginx:1.27 --replicas=2
kubectl get pods -o wide                                  # note IP and NODE

# 2. Pod to Pod by IP
kubectl run client --image=busybox:1.36 -- sleep 3600
IP=$(kubectl get pod -l app=web -o jsonpath='{.items[0].status.podIP}')
kubectl exec client -- wget -qO- http://$IP | head -4     # the nginx welcome page

# 3. What the Pod sees
kubectl exec client -- ip addr                            # eth0 has the Pod IP
kubectl exec client -- ip route                           # default route via the node bridge
kubectl exec client -- cat /etc/resolv.conf               # DNS server and search domains

# 4. Address ranges
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD_CIDR:.spec.podCIDR
kubectl get svc kubernetes                                # a ClusterIP from the Service range

# 5. Containers in one Pod share localhost
kubectl apply -f shared-net.yaml
kubectl exec shared-net -c probe -- wget -qO- http://localhost | head -4

# 6. Pod IPs are ephemeral
kubectl get pods -l app=web -o wide
kubectl delete pod -l app=web
kubectl get pods -l app=web -o wide                       # new Pods, new IPs

# Cleanup
kubectl delete deployment web
kubectl delete pod client shared-net
```

With a multi-node cluster (`kind` with 3 nodes), check that two Pods on **different nodes** can reach each other too.

## Gotchas

- **Never hard-code Pod IPs.** They change whenever a Pod is recreated. Use a Service.
- `hostNetwork: true` puts the Pod on the node's network. It then uses the node's IP and ports and can clash with node processes.
- Two containers in one Pod **cannot listen on the same port**.
- On EKS with the VPC CNI, the number of Pod IPs per node is limited by the instance type's network interfaces, so a small instance may run out of IPs before CPU.
- A `NetworkPolicy` that "does nothing" often means the CNI does not enforce policies.

## Check yourself

1. What does it mean that Pods communicate "without NAT"?
2. Who implements the network model, Kubernetes or the CNI plugin?
3. Why can two containers in a Pod use `localhost` to talk?
4. Name the three address ranges in a cluster.
5. Why should you not rely on a Pod's IP?

<details>
<summary>Answers</summary>

1. The receiver sees the real source Pod IP. Addresses are not rewritten between Pods
2. The CNI plugin. Kubernetes only defines the rules
3. They share one network namespace (the Pod sandbox)
4. Node IPs, Pod CIDR, Service CIDR
5. It changes every time the Pod is recreated or rescheduled

</details>

---

[README](README.md) | [Services →](services/README.md)