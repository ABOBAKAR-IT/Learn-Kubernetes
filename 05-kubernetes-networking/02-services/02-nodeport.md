# NodePort

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/03-service/manifests/`](../../labs/03-service/manifests/) · Worked example: [`labs/03-service`](../../labs/03-service/)

[← ClusterIP](clusterip.md) | [Module README](../README.md) | [LoadBalancer →](loadbalancer.md)

---

## Idea

A **NodePort** Service opens the **same port on every node** (default range **30000-32767**). Traffic to `<any-node-IP>:<nodePort>` is forwarded to the Service and then to a Pod.

```
outside ─► NodeIP:30081 ──► Service (ClusterIP created automatically) ──► Pod :8080
           on EVERY node        port 80                                    targetPort
```

- NodePort **includes a ClusterIP**, so the Service still works inside the cluster.
- It works on **every node**, even a node that runs no matching Pod (kube-proxy forwards the traffic).
- It is the simplest way to reach a Service from outside a local cluster.

## YAML explained

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-nodeport
spec:
  type: NodePort
  selector:
    app: echo
  ports:
    - name: http
      port: 80            # Service port (inside the cluster)
      targetPort: 8080    # Pod port
      nodePort: 30081     # port on every node; omit it to get a random free one
```

The full chain:

```
nodePort 30081 (every node) → port 80 (Service) → targetPort 8080 (Pod) → app
```

## Traffic policy: where does the packet go?

```yaml
spec:
  externalTrafficPolicy: Cluster   # default
```

| Policy | Behavior | Client IP | Extra hop |
|---|---|---|---|
| `Cluster` (default) | Any node forwards to any Pod in the cluster | **Lost** (source NAT) | Possible |
| `Local` | A node forwards only to Pods **on that node** | **Preserved** | No |

With `Local`, a node that has no matching Pod does not answer, so a load balancer must health-check nodes.

## Reaching it

| Cluster | How |
|---|---|
| Real cluster or cloud | `http://<node-IP>:30081` (open the port in the firewall or security group) |
| minikube | `minikube service echo-nodeport` (or `--url`). With the Docker driver on macOS or Windows the node IP is not directly reachable |
| kind | Map the port when creating the cluster (`extraPortMappings`), or use `kubectl port-forward` |

```bash
kubectl get nodes -o wide        # INTERNAL-IP / EXTERNAL-IP of each node
```

## Lab

```bash
cd labs/03-service/manifests
kubectl apply -f echo-backend.yaml
kubectl apply -f nodeport.yaml

kubectl get svc echo-nodeport                       # PORT(S) 80:30081/TCP
kubectl get endpointslices -l kubernetes.io/service-name=echo-nodeport
minikube service echo-nodeport --url                # prints the reachable URL
curl $(minikube service echo-nodeport --url)
```

**Explore**

```bash
# NodePort also created a ClusterIP: it works from inside the cluster
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://echo-nodeport | head -5

# Random node port: remove nodePort from the YAML and re-apply
kubectl get svc echo-nodeport                       # a new number in 30000-32767

# Rules kube-proxy wrote on a node
minikube ssh
sudo iptables -t nat -L KUBE-NODEPORTS -n | grep 30
exit

# Client IP preservation (watch the "client_address" / remote address in the response)
kubectl patch svc echo-nodeport -p '{"spec":{"externalTrafficPolicy":"Local"}}'
curl $(minikube service echo-nodeport --url)
kubectl patch svc echo-nodeport -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

**Cleanup**

```bash
kubectl delete -f nodeport.yaml -f echo-backend.yaml
```

## When to use NodePort

| Good for | Not good for |
|---|---|
| Local development and demos | Production traffic directly to nodes |
| Putting your own load balancer in front of the nodes | Many services (each uses a cluster-wide port) |
| Quick debugging | Anything needing a stable public address or TLS |

For production, use a [LoadBalancer](loadbalancer.md) or an [Ingress](../ingress/README.md).

## Gotchas

- The port must be **free and in range**. A conflict is rejected: `provided port is already allocated`.
- Each NodePort uses a port **across the whole cluster**. The range is finite.
- Exposing NodePorts means opening **node-level firewall ports**. Restrict the source addresses.
- Node IPs can change (autoscaling, replacement), so clients should not depend on one node.
- With `externalTrafficPolicy: Cluster`, the real client IP is hidden from your app.

## Check yourself

1. On which nodes does a NodePort listen?
2. What range of ports does it use by default?
3. Does a NodePort Service still have a ClusterIP?
4. What changes when `externalTrafficPolicy: Local` is set?
5. Why is NodePort rarely used directly in production?

<details>
<summary>Answers</summary>

1. On every node in the cluster
2. 30000-32767
3. Yes, it is created automatically
4. The node forwards only to local Pods and the client IP is preserved. Nodes without a Pod do not answer
5. It exposes node ports, uses scarce cluster-wide ports and gives no stable address or TLS handling

</details>

---

[← ClusterIP](clusterip.md) | [Module README](../README.md) | [LoadBalancer →](loadbalancer.md)