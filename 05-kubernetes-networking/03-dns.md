# DNS in Kubernetes

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/03-service/manifests/`](../labs/03-service/manifests/)

[← LoadBalancer](services/loadbalancer.md) | [README](README.md) | [Ingress →](ingress/README.md)

---

## Idea

Every Service gets a **DNS name** automatically, so Pods call `http://echo` instead of an IP. The DNS server inside the cluster is **CoreDNS**.

```
Pod ──"echo"──► CoreDNS (Service kube-dns, e.g. 10.96.0.10) ──► 10.96.14.20 (ClusterIP of echo)
```

## The pieces

| Piece | Where | Job |
|---|---|---|
| **CoreDNS** | Deployment `coredns` in `kube-system` | Answers cluster DNS queries |
| **kube-dns Service** | `kube-system` (stable ClusterIP) | The address Pods use for DNS |
| **`/etc/resolv.conf`** | Written by the kubelet into every Pod | Points the Pod at CoreDNS and sets search domains |

```bash
kubectl -n kube-system get deploy coredns
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

## What a Pod's `resolv.conf` looks like

```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

- `nameserver`: the kube-dns ClusterIP.
- `search`: suffixes tried for short names. The first one is the Pod's **own namespace**.
- `ndots:5`: a name with **fewer than 5 dots** is tried with each search suffix **before** being tried as-is.

## DNS names you get

| Object | Name | Returns |
|---|---|---|
| Service | `<svc>.<ns>.svc.cluster.local` | The ClusterIP |
| Headless Service | `<svc>.<ns>.svc.cluster.local` | The IPs of all Ready Pods |
| StatefulSet Pod | `<pod>.<svc>.<ns>.svc.cluster.local` | That Pod's IP (for example `web-0.web...`) |
| Named port | `_<port>._<proto>.<svc>.<ns>.svc.cluster.local` (SRV) | Port number and host |
| Pod | `10-244-1-5.<ns>.pod.cluster.local` | The Pod IP (dashes replace dots) |

`cluster.local` is the default cluster domain and can be changed.

### Short names from different places

```
Same namespace:     echo
Other namespace:    echo.shop
Fully qualified:    echo.shop.svc.cluster.local.     (trailing dot = absolute name)
```

From a Pod in `default`, the name `echo` is expanded using the search list, so the first try is `echo.default.svc.cluster.local`, which succeeds.

## The `ndots:5` surprise

An external name like `api.example.com` has only 2 dots, which is fewer than 5. The resolver first tries:

```
api.example.com.default.svc.cluster.local   → NXDOMAIN
api.example.com.svc.cluster.local           → NXDOMAIN
api.example.com.cluster.local               → NXDOMAIN
api.example.com                             → answer
```

That is several wasted queries for every external lookup (each often for both IPv4 and IPv6). It adds latency and load on CoreDNS.

Fixes:

```yaml
# 1. Use a fully qualified name with a trailing dot in your app config:  api.example.com.
# 2. Lower ndots for the Pod:
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```

## `dnsPolicy`

| Value | Behavior |
|---|---|
| `ClusterFirst` (default) | Use CoreDNS for cluster names, forward everything else to the node's resolvers |
| `ClusterFirstWithHostNet` | Same, for Pods with `hostNetwork: true` |
| `Default` | Inherit the **node's** DNS settings (no cluster names) |
| `None` | Ignore defaults, use only what `dnsConfig` provides |

## CoreDNS configuration

CoreDNS reads its **Corefile** from a ConfigMap:

```bash
kubectl -n kube-system get configmap coredns -o yaml
```

Typical contents:

```
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

| Plugin | Job |
|---|---|
| `kubernetes` | Answers cluster names from the Service and Pod data in the API |
| `forward` | Sends everything else upstream (the node's resolvers) |
| `cache` | Caches answers (30 s here) |
| `loop` | Detects forwarding loops and stops CoreDNS |
| `reload` | Applies Corefile changes without restart |

Send one company domain to a private resolver by adding a block:

```
corp.example.com:53 {
    forward . 10.0.0.2
}
```

Large clusters often add **NodeLocal DNSCache**, a per-node cache that reduces load and latency.

## Lab

```bash
cd labs/03-service/manifests
kubectl apply -f echo-backend.yaml -f clusterip.yaml -f headless.yaml
kubectl apply -f dns-debug.yaml                       # a Pod with dig and nslookup
kubectl wait --for=condition=Ready pod/dnsutils --timeout=90s
```

**1. Look at the Pod's resolver settings**

```bash
kubectl exec dnsutils -- cat /etc/resolv.conf
```

**2. Resolve Service names**

```bash
kubectl exec dnsutils -- nslookup kubernetes.default            # the API server Service
kubectl exec dnsutils -- nslookup echo                          # short name
kubectl exec dnsutils -- nslookup echo.default.svc.cluster.local
kubectl exec dnsutils -- nslookup echo-headless                 # returns every Pod IP
kubectl get pods -l app=echo -o wide                            # compare with the Pod IPs above
```

**3. SRV and Pod records**

```bash
kubectl exec dnsutils -- dig +short SRV _http._tcp.echo.default.svc.cluster.local
POD_IP_DASHED=$(kubectl get pod -l app=echo -o jsonpath='{.items[0].status.podIP}' | tr . -)
kubectl exec dnsutils -- nslookup $POD_IP_DASHED.default.pod.cluster.local
```

**4. See `ndots` at work**

```bash
kubectl exec dnsutils -- dig +search +showsearch example.com | head -20
kubectl exec dnsutils -- dig +short example.com.                # trailing dot: one query
```

**5. Across namespaces**

```bash
kubectl create namespace other
kubectl create deployment web -n other --image=nginx:1.27
kubectl expose deployment web -n other --port=80
kubectl exec dnsutils -- nslookup web                           # fails: wrong namespace
kubectl exec dnsutils -- nslookup web.other                     # works
```

**6. Break DNS on purpose, then fix it**

```bash
kubectl -n kube-system get deploy coredns                       # note the current replica count (R)
kubectl -n kube-system scale deploy coredns --replicas=0
kubectl exec dnsutils -- nslookup echo                          # fails
IP=$(kubectl get svc echo -o jsonpath='{.spec.clusterIP}')
kubectl exec dnsutils -- wget -qO- -T 3 http://$IP | head -3    # still works by IP
kubectl -n kube-system scale deploy coredns --replicas=R        # restore the original count
```

**Cleanup**

```bash
kubectl delete -f dns-debug.yaml -f headless.yaml -f clusterip.yaml -f echo-backend.yaml
kubectl delete namespace other
```

## Troubleshooting DNS

| Symptom | Likely cause | Check |
|---|---|---|
| `NXDOMAIN` for a Service | Wrong name or namespace, Service missing | `kubectl get svc -A`, use `name.namespace` |
| Lookups time out | CoreDNS down or unreachable, NetworkPolicy blocking port 53 | `kubectl -n kube-system get pods -l k8s-app=kube-dns`, check egress policies |
| External names slow | `ndots:5` expansions | Trailing dot, or lower `ndots` |
| Works by IP, not by name | DNS problem, not networking | Compare `nslookup` with `wget` to the IP |
| Pod uses wrong resolver | `dnsPolicy: Default` or `hostNetwork` | `kubectl get pod -o yaml \| grep -i dnspolicy` |

```bash
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=30
```

## Gotchas

- DNS answers a Service name even when it has **no endpoints**. Name resolution working does not mean traffic works.
- A **NetworkPolicy with default-deny egress** breaks DNS unless you allow port 53 to CoreDNS.
- A Pod only resolves short names in **its own namespace**. Add the namespace for others.
- Do not store the resolved IP in app config, it is the whole point of the name.

## Check yourself

1. Which component answers DNS queries in the cluster?
2. What is the full DNS name of Service `echo` in namespace `shop`?
3. What does `ndots:5` cause for `api.example.com`?
4. What does DNS return for a headless Service?
5. A Pod can reach a Service by IP but not by name. Where do you look?

<details>
<summary>Answers</summary>

1. CoreDNS (reached through the `kube-dns` Service)
2. `echo.shop.svc.cluster.local`
3. Several failing lookups with the search suffixes appended before the real name is tried
4. The IPs of all Ready Pods
5. CoreDNS health and logs, `/etc/resolv.conf` in the Pod, and any NetworkPolicy blocking port 53

</details>

---

[← LoadBalancer](services/loadbalancer.md) | [README](README.md) | [Ingress →](ingress/README.md)