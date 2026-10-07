# kube-proxy

> Status: 📘 Notes ready

## Idea

**kube-proxy** makes **Services** work. A Service has a virtual IP that no machine owns. kube-proxy runs on every node and programs the node's network rules so traffic to that IP is sent to a real Pod.

- It watches **Services** and **EndpointSlices** in the API server.
- On every change it updates rules on its node.
- In the default modes it is **not in the data path**: it only writes rules, and the **Linux kernel** does the forwarding.

```
Pod A ──► Service IP 10.96.0.50:80
              │  (kernel rules written by kube-proxy: DNAT)
              ├──► Pod 10.244.1.4:8080   ← chosen
              ├──► Pod 10.244.2.7:8080
              └──► Pod 10.244.2.9:8080
```

## Modes

| Mode | How | Notes |
|---|---|---|
| **iptables** (default) | Chains of iptables rules, random choice of Pod | Simple, common; slows with many thousands of Services |
| **IPVS** | Kernel load balancer with more algorithms | Better for very large clusters |
| **nftables** | Newer rule backend | Intended successor to iptables mode |

Some network plugins (for example Cilium) can replace kube-proxy completely.

## How it runs

In kubeadm-style clusters it is a **DaemonSet** in `kube-system`, with its settings in a ConfigMap. This is the DaemonSet idea from the workloads module: one Pod per node.

## What it does NOT do

- It does **not** route Pod-to-Pod traffic. That is the **CNI plugin** (Calico, Cilium, AWS VPC CNI).
- It does **not** provide DNS. That is **CoreDNS**.
- It does **not** create load balancers in the cloud. That is the cloud controller.

## Lab

```bash
# 1. Find kube-proxy
kubectl -n kube-system get ds kube-proxy
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide      # one per node

# 2. Which mode?
kubectl -n kube-system get cm kube-proxy -o yaml | grep -E "mode:|^  *mode"

# 3. Make a Service
kubectl create deployment web --image=nginx:1.27 --replicas=2
kubectl expose deployment web --port=80
kubectl get svc web
kubectl get endpointslices -l kubernetes.io/service-name=web

# 4. See the rules kube-proxy wrote (on a node)
minikube ssh
sudo iptables -t nat -L KUBE-SERVICES -n | grep -i web
sudo iptables-save -t nat | grep -E "KUBE-SVC|KUBE-SEP" | head
exit
```

You will see a `KUBE-SVC-...` chain for the Service and `KUBE-SEP-...` chains, one per Pod endpoint, each doing a DNAT to a Pod IP.

```bash
# 5. Scale and watch the endpoints change (kube-proxy updates the rules)
kubectl scale deployment web --replicas=4
kubectl get endpointslices -l kubernetes.io/service-name=web -o wide

# Cleanup
kubectl delete svc web
kubectl delete deployment web
```

## Gotchas

- A Service whose selector matches no Ready Pods has **no endpoints**, so traffic fails even though kube-proxy is healthy.
- If kube-proxy is down on a node, **new Service changes do not reach that node**, and existing rules keep working until they go stale.
- `kubectl get svc` showing a ClusterIP does not mean it answers `ping`. It only handles the Service's ports.

## Check yourself

1. What does kube-proxy watch?
2. Does kube-proxy forward the packets itself in iptables mode?
3. Which component handles Pod-to-Pod networking?
4. How does kube-proxy usually run?
5. A Service has no endpoints. Is kube-proxy the problem?

<details>
<summary>Answers</summary>

1. Services and EndpointSlices
2. No. It writes rules and the kernel forwards the traffic
3. The CNI plugin
4. As a DaemonSet, one Pod per node
5. Usually not. The Service selector matches no Ready Pods

</details>