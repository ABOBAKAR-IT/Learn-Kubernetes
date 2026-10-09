# LoadBalancer

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/03-service/manifests/`](../../labs/03-service/manifests/) · Worked example: [`labs/03-service`](../../labs/03-service/)

[← NodePort](nodeport.md) | [Module README](../README.md) | [DNS →](../dns.md)

---

## Idea

A **LoadBalancer** Service asks the environment for a real **external load balancer** and gives the Service a public (or private) address in `EXTERNAL-IP`.

```
Internet ─► Cloud load balancer ─► NodePort on the nodes ─► Service ─► Pod
            (EXTERNAL-IP)
```

- It **includes NodePort and ClusterIP**. The load balancer sends traffic to the node ports.
- The **cloud controller manager** creates it, and deleting the Service deletes it.
- On a cluster with no cloud integration, `EXTERNAL-IP` stays `<pending>` forever.

## YAML explained

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-lb
spec:
  type: LoadBalancer
  selector:
    app: echo
  ports:
    - name: http
      port: 80            # the load balancer's listener port
      targetPort: 8080    # the Pod port
  # optional
  # loadBalancerSourceRanges: ["203.0.113.0/24"]   # only these client CIDRs may connect
  # externalTrafficPolicy: Local                     # preserve client IP, skip extra hop
```

```
client → LB :80 → NodePort (auto) → Service :80 → Pod :8080
```

## Where does the load balancer come from?

| Environment | What provides it |
|---|---|
| **AWS (EKS)** | The in-tree cloud controller (Classic Load Balancer) or the **AWS Load Balancer Controller** (Network Load Balancer) |
| GCP, Azure | The provider's cloud controller |
| **minikube** | `minikube tunnel` fakes one on your machine |
| **kind** and bare metal | Add an implementation such as **MetalLB** (or cloud-provider-kind) |

### On AWS EKS

With the **AWS Load Balancer Controller** installed, you choose a Network Load Balancer with annotations:

```yaml
metadata:
  name: echo-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip      # or: instance
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing  # or: internal
spec:
  type: LoadBalancer
```

Newer versions also support selecting the controller with `spec.loadBalancerClass: service.k8s.aws/nlb`. Annotation names and defaults change between controller versions, so check the AWS Load Balancer Controller documentation for the version you run.

- `target-type: ip` sends traffic straight to Pod IPs (works well with the VPC CNI).
- `instance` sends traffic to node ports.
- An `internal` scheme gives a private load balancer inside your VPC.

## LoadBalancer vs Ingress

| | One LoadBalancer per Service | One Ingress for many Services |
|---|---|---|
| Layer | 4 (TCP/UDP) | 7 (HTTP/HTTPS) |
| Cost | **One load balancer per Service** | One load balancer for many apps |
| Routing | None, one Service per address | Host and path rules, TLS |
| Good for | Non-HTTP protocols, a single public service | Many web apps and APIs |

See [Ingress](../ingress/README.md).

## Lab

```bash
cd labs/03-service/manifests
kubectl apply -f echo-backend.yaml
kubectl apply -f loadbalancer.yaml

kubectl get svc echo-lb -w                      # EXTERNAL-IP is <pending>
```

Open a **second terminal** and run (it needs sudo and must stay running):

```bash
minikube tunnel
```

Back in the first terminal:

```bash
kubectl get svc echo-lb                         # EXTERNAL-IP now has an address
curl http://<EXTERNAL-IP>                       # often 127.0.0.1 or the Service IP, depending on driver
```

**Explore**

```bash
kubectl describe svc echo-lb                    # Events show the load balancer being ensured
kubectl get svc echo-lb -o yaml | grep -A3 nodePort   # the NodePort created underneath
kubectl patch svc echo-lb -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

**Cleanup** (stop `minikube tunnel` with Ctrl+C)

```bash
kubectl delete -f loadbalancer.yaml -f echo-backend.yaml
```

## Gotchas

- **Cost:** every `type: LoadBalancer` Service creates a billable cloud load balancer. Ten services means ten load balancers. Put web apps behind one Ingress instead.
- **`<pending>` forever** means no controller is providing load balancers.
- Deleting the Service **deletes the cloud load balancer** and its address. A recreated Service gets a new address unless you reserve one.
- Cloud load balancers have **idle timeouts** and their own health checks. Long-lived connections may need tuning.
- Restrict access with `loadBalancerSourceRanges` or security groups. A public load balancer is open to the internet by default.

## Check yourself

1. What does a LoadBalancer Service include besides the cloud load balancer?
2. Why is `EXTERNAL-IP` `<pending>` on minikube, and how do you fix it?
3. What is the cost problem with one LoadBalancer per Service?
4. How do you request an internal (private) load balancer on AWS?
5. When would you choose a LoadBalancer over an Ingress?

<details>
<summary>Answers</summary>

1. A NodePort and a ClusterIP
2. There is no cloud provider. Run `minikube tunnel`
3. Each Service creates its own billable load balancer
4. The `aws-load-balancer-scheme: internal` annotation (with the AWS Load Balancer Controller)
5. For non-HTTP traffic (raw TCP/UDP) or a single public service

</details>

---

[← NodePort](nodeport.md) | [Module README](../README.md) | [DNS →](../dns.md)