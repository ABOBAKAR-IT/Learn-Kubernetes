# Ingress

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/05-ingress/manifests/`](../../labs/05-ingress/manifests/) · Lab: [`labs/05-ingress`](../../labs/05-ingress/)

[← DNS](../dns.md) | [Module README](../README.md) | [Network Policies →](../network-policies/README.md)

---

> **Status check (important):** the Kubernetes project announced the **retirement of Ingress NGINX**, with best-effort maintenance only until **March 2026**, and no releases, bug fixes or security fixes after that. Existing installations keep working, but **do not choose it for new production clusters**. The Ingress concepts below are the same for every controller, so learn them here, then pick a maintained controller (see [Choosing a controller](#choosing-a-controller)) or move to [Gateway API](#gateway-api-the-successor).

## Idea

An **Ingress** routes **HTTP and HTTPS** traffic from outside the cluster to Services, using rules on **hostnames and paths**. One entry point can serve many apps.

```
Internet
   │
   ▼
Cloud load balancer  (one address, one bill)
   │
   ▼
Ingress controller Pods  (the actual proxy: reads Ingress rules)
   │   host: shop.example.com   path /api  ──► Service api ──► Pods
   │   host: shop.example.com   path /     ──► Service web ──► Pods
   └   host: admin.example.com  path /     ──► Service admin ──► Pods
```

## Two parts you must have

| Part | What it is |
|---|---|
| **Ingress** (the resource) | Your **rules**: a YAML object. On its own it does **nothing** |
| **Ingress controller** | The **software** (a proxy running in Pods) that reads the rules and routes traffic |

No controller means no routing: `kubectl get ingress` shows an empty ADDRESS. An **IngressClass** names which controller handles an Ingress, selected with `ingressClassName`.

```bash
kubectl get ingressclass
kubectl get pods -A | grep -i ingress
```

## YAML explained

**Path-based routing**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
spec:
  ingressClassName: nginx            # which controller handles this
  rules:
    - host: demo.local
      http:
        paths:
          - path: /a
            pathType: Prefix
            backend:
              service:
                name: app-a
                port:
                  number: 80
          - path: /b
            pathType: Prefix
            backend:
              service:
                name: app-b
                port:
                  number: 80
```

**Host-based routing**

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: a.demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: app-a, port: { number: 80 } }
    - host: b.demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: app-b, port: { number: 80 } }
```

**TLS (HTTPS)**

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - demo.local
      secretName: demo-tls            # a Secret of type kubernetes.io/tls
  rules:
    - host: demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: app-a, port: { number: 80 } }
```

```bash
kubectl create secret tls demo-tls --cert=tls.crt --key=tls.key
```

**Default backend** (anything that matches no rule):

```yaml
spec:
  defaultBackend:
    service:
      name: app-a
      port:
        number: 80
```

### Path types

| `pathType` | Matches |
|---|---|
| `Exact` | Only that exact path. `/a` matches `/a`, not `/a/` or `/a/x` |
| `Prefix` | The path and everything under it, **by path segments**. `/a` matches `/a`, `/a/`, `/a/x`, **not** `/ab` |
| `ImplementationSpecific` | Up to the controller (often regex) |

When several rules match, the **longest** path wins.

## How a request travels

1. DNS maps `demo.local` to the controller's load balancer address.
2. The request reaches the **controller Pod**.
3. The controller finds the rule by **Host header and path**.
4. It forwards to a Pod of the matching Service (usually directly to Pod IPs, not through the Service IP).
5. If no rule matches: the controller's **default backend** returns `404`. If the Service has no ready Pods: `503`.

## Controller-specific features: annotations

Anything beyond the basics (rewrites, timeouts, redirects, authentication) is configured with **annotations that differ by controller**. This makes Ingress YAML **not fully portable**.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /       # example for the NGINX controller
```

## Choosing a controller

| Controller | Notes |
|---|---|
| **AWS Load Balancer Controller** | Creates an **ALB** per Ingress (or one shared ALB with a group). Natural choice on EKS |
| **Traefik** | Popular, simple, supports Ingress and Gateway API |
| **HAProxy Ingress**, **Contour** (Envoy) | Mature, well maintained |
| **NGINX Inc. "nginx-ingress"** | A different project from the retired community Ingress NGINX |
| Ingress NGINX (community) | **Retired.** Fine for a throwaway learning cluster only |

On EKS with the AWS Load Balancer Controller, typical annotations (check the controller docs for your version):

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...      # HTTPS with an ACM certificate
    alb.ingress.kubernetes.io/group.name: shared                    # share one ALB across Ingresses
spec:
  ingressClassName: alb
```

## Automatic certificates: cert-manager

**cert-manager** requests and renews certificates (for example from Let's Encrypt) and writes them into the TLS Secret. You add an annotation such as `cert-manager.io/cluster-issuer: letsencrypt-prod` to the Ingress, and it does the rest.

## Gateway API: the successor

The **Gateway API** is the newer, more expressive model that the Kubernetes project recommends for new work. It separates roles and is not tied to annotations.

| Object | Owned by | Meaning |
|---|---|---|
| `GatewayClass` | Infrastructure provider | Which implementation |
| `Gateway` | Platform team | The listeners (ports, protocols, TLS) |
| `HTTPRoute` | App team | Routing rules to Services |

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
spec:
  gatewayClassName: <class from your implementation>
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
spec:
  parentRefs:
    - name: demo-gateway
  hostnames: ["demo.local"]
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /a
      backendRefs:
        - name: app-a
          port: 80
```

The Gateway API CRDs are installed separately, and you need an **implementation** (for example Envoy Gateway, Cilium, Istio, Traefik, NGINX Gateway Fabric). Ingress itself is not going away, but it receives no new features.

## Lab

The full lab, with manifests, is in [`labs/05-ingress`](../../labs/05-ingress/). In short:

```bash
minikube addons enable ingress                 # installs a controller for learning
kubectl get pods -n ingress-nginx
cd labs/05-ingress/manifests
kubectl apply -f apps.yaml                     # app-a and app-b with Services
kubectl apply -f ingress-path.yaml
curl -H "Host: demo.local" http://$(minikube ip)/a     # hello from app-a
curl -H "Host: demo.local" http://$(minikube ip)/b     # hello from app-b
```

With the Docker driver on macOS or Windows, run `minikube tunnel` and use `http://127.0.0.1` instead of `minikube ip`. Because the minikube addon is based on the retired controller, treat it as a classroom tool. The lab README shows how to repeat it with Traefik.

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| ADDRESS empty | No controller, or wrong `ingressClassName` | `kubectl get ingressclass`, controller Pods |
| `404` from the controller | Host or path does not match any rule | `kubectl describe ingress`, curl with the right `Host` header |
| `503 Service Temporarily Unavailable` | Service has no ready endpoints | `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` |
| `502 Bad Gateway` | Pod crashes or `targetPort` wrong | Pod logs, Service ports |
| Certificate warning | Self-signed or wrong host in the cert | `curl -vk`, TLS Secret contents |
| Rule ignored | Missing or wrong annotation for that controller | Controller docs and logs |

```bash
kubectl describe ingress demo
kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=30
```

## Gotchas

- An Ingress with **no controller does nothing**, and there is no error.
- Ingress handles **HTTP/HTTPS only**. For other protocols use a LoadBalancer Service.
- The Ingress and the backend Services must be in the **same namespace**.
- `Prefix` matches whole path segments, so `/a` does not match `/ab`.
- Annotations are **controller-specific**, so moving controllers means rewriting them.
- Put the **Host** in your test requests, otherwise the controller returns 404.

## Check yourself

1. What is the difference between an Ingress and an Ingress controller?
2. Which protocols does Ingress handle?
3. What is the difference between `Exact` and `Prefix` path types?
4. How is TLS configured on an Ingress?
5. Why is one Ingress cheaper than many LoadBalancer Services?
6. What is the Gateway API, and why does the project recommend it?

<details>
<summary>Answers</summary>

1. The Ingress is the rules object. The controller is the proxy software that enforces them. Without a controller nothing happens
2. HTTP and HTTPS
3. `Exact` matches only that path. `Prefix` matches the path and everything below it by path segments
4. A `tls` block with hostnames and a `kubernetes.io/tls` Secret (`secretName`)
5. One shared load balancer and controller serve many hosts and paths
6. A newer routing API with separate roles (GatewayClass, Gateway, HTTPRoute) and less reliance on annotations. Ingress is feature-frozen

</details>

---

[← DNS](../dns.md) | [Module README](../README.md) | [Network Policies →](../network-policies/README.md)