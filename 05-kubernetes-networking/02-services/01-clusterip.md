# ClusterIP

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/03-service/manifests/`](../../labs/03-service/manifests/)

[← Services](README.md) | [Module README](../README.md) | [NodePort →](nodeport.md)

---

## Idea

**ClusterIP** is the **default** Service type. It gives the Service a virtual IP that is reachable **only inside the cluster**. This is how most services talk to each other: frontend → API → database.

```
Pod (frontend) ──► http://echo  ──► ClusterIP 10.96.14.20:80 ──► one of the echo Pods :8080
```

- The IP comes from the **Service CIDR** and never changes for the life of the Service.
- It is reachable by **name** through DNS (`echo`, `echo.default`, `echo.default.svc.cluster.local`).
- It is not reachable from outside the cluster.

## YAML explained

First the backend the Service points at:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: echo
          image: kicbase/echo-server:1.0
          ports:
            - name: http
              containerPort: 8080
```

Then the Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  type: ClusterIP            # optional, it is the default
  selector:
    app: echo                # matches the Pod labels above
  ports:
    - name: http
      port: 80               # clients call echo:80
      targetPort: http       # the container port NAMED "http" (8080)
```

| Field | Meaning |
|---|---|
| `selector` | Which Pods receive traffic (label match, Ready only) |
| `port` | The port on the Service |
| `targetPort` | The Pod's port, by number or by **name**. A name survives a port change in the Deployment |
| `name` (on the port) | Required when there are several ports |

Several ports in one Service:

```yaml
ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
```

## Lab

```bash
cd labs/03-service/manifests
kubectl apply -f echo-backend.yaml
kubectl apply -f clusterip.yaml
kubectl get svc echo
kubectl get endpointslices -l kubernetes.io/service-name=echo     # 3 Pod IPs
```

**1. Call it by name from inside the cluster**

```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://echo
```

**2. See the load balancing**

```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh -c '
  for i in 1 2 3 4 5 6; do wget -qO- http://echo | grep -i hostname; done'
```

You should see different Pod names across the six calls.

**3. Try it from outside (it will not work directly)**

```bash
kubectl get svc echo                         # ClusterIP only, no external address
kubectl port-forward svc/echo 8080:80        # a debugging tunnel from your laptop
curl http://localhost:8080
```

`port-forward` is for debugging. It is not a way to expose a service.

**4. Watch endpoints react**

```bash
kubectl scale deployment echo --replicas=5
kubectl get endpointslices -l kubernetes.io/service-name=echo -o wide     # 5 IPs
kubectl scale deployment echo --replicas=3
```

**5. Break the link on purpose**

```bash
kubectl patch svc echo -p '{"spec":{"selector":{"app":"wrong"}}}'
kubectl get endpointslices -l kubernetes.io/service-name=echo             # no endpoints
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://echo   # times out
kubectl apply -f clusterip.yaml                                           # restore
```

**6. Sticky sessions**

```bash
kubectl patch svc echo -p '{"spec":{"sessionAffinity":"ClientIP"}}'
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh -c '
  for i in 1 2 3 4 5 6; do wget -qO- http://echo | grep -i hostname; done'   # same Pod every time
kubectl patch svc echo -p '{"spec":{"sessionAffinity":"None"}}'
```

**Cleanup**

```bash
kubectl delete -f clusterip.yaml -f echo-backend.yaml
```

## Gotchas

- The Service IP **does not answer ping**. Test with the actual port (`wget`, `curl`, `nc`).
- `targetPort` must match what the app listens on, not what `containerPort` says.
- A Service only sends traffic to **Ready** Pods. A failing readiness probe means no traffic.
- To expose a ClusterIP service to the outside you need an Ingress, NodePort or LoadBalancer in front of it.

## Check yourself

1. Who can reach a ClusterIP Service?
2. Which three DNS names resolve to the same Service from another namespace?
3. Why use a named `targetPort`?
4. How do you test a ClusterIP Service from your laptop for debugging?
5. What does `sessionAffinity: ClientIP` do?

<details>
<summary>Answers</summary>

1. Only things inside the cluster (Pods, and nodes)
2. From another namespace: `echo.default`, and `echo.default.svc`, and `echo.default.svc.cluster.local`
3. The Service keeps working if the container port number changes in the Deployment
4. `kubectl port-forward svc/<name> <local>:<port>`
5. Sends repeated requests from the same client IP to the same Pod

</details>

---

[← Services](README.md) | [Module README](../README.md) | [NodePort →](nodeport.md)