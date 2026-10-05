# Kubernetes Lab: Welcome App (Pod, Deployment, NodePort, LoadBalancer)

A small hands-on lab that runs an echo web server on **minikube** and exposes it in two ways.
It teaches how **Pod → Deployment → Service** depend on each other.

## Files in this lab

| File | Kind | Purpose |
|---|---|---|
| `pod.yaml` | Pod | One bare Pod (learning only, no self-healing) |
| `deployment.yaml` | Deployment | Runs and manages 2 Pods (the real way) |
| `service-nodeport.yaml` | Service (NodePort) | Exposes the Pods on a node port (dev access) |
| `loadbalancer.yaml` | Service (LoadBalancer) | Exposes the Pods through a load balancer |

---

## 1. Big picture

```
 You (browser / curl)
        │
        ▼
 ┌──────────────────────────────┐
 │ Service (welcome-service)    │  stable IP + DNS name, load balances
 │ selector: app=welcome        │
 └──────────┬───────────────────┘
            │ finds Pods by LABEL  app=welcome  (only Ready Pods)
     ┌──────┴───────┐
     ▼              ▼
  Pod (8080)     Pod (8080)          ← created and kept alive by
     ▲              ▲                   Deployment → ReplicaSet
     └──────┬───────┘
     Deployment (welcome-deployment, replicas: 2)
```

**One rule connects everything: the label `app: welcome`.**

- The Deployment `selector` finds its Pods by `app: welcome`.
- The Deployment `template` stamps `app: welcome` on every Pod it creates.
- The Service `selector` finds Pods to send traffic to by `app: welcome`.

If those labels do not match, nothing connects.

---

## 2. Service types (theory)

| Type | Meaning |
|---|---|
| **Service** | Abstraction that defines a logical set of Pods and a policy to access them. The Pods are usually chosen by a selector. |
| **ClusterIP** (default) | Exposes the Service on a cluster-internal IP. Reachable only from inside the cluster. |
| **NodePort** | Exposes the Service on every Node's IP at a static port. A ClusterIP is created automatically. Reach it at `<NodeIP>:<NodePort>`. |
| **LoadBalancer** | Exposes the Service externally using a cloud provider's load balancer. NodePort and ClusterIP are created automatically. |
| **ExternalName** | Maps the Service to an external DNS name (CNAME). No proxying is set up. |

Each type builds on the previous one:

```
LoadBalancer ⊃ NodePort ⊃ ClusterIP
```

---

## 3. The YAML files, explained

Every Kubernetes object has the same 4 top-level keys:

```yaml
apiVersion:   # which API group and version defines this kind
kind:         # type of object (case sensitive: Pod, Deployment, Service)
metadata:     # name, labels, namespace
spec:         # the desired state
```

### 3.1 `pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: welcome-pod
  labels:
    app: welcome
spec:
  containers:
    - name: welcome-container
      image: kicbase/echo-server:1.0
      ports:
        - containerPort: 8080
```

| Line | Meaning |
|---|---|
| `apiVersion: v1` | Pods live in the core API group, version `v1` |
| `kind: Pod` | Create a Pod |
| `metadata.name` | Unique Pod name in the namespace |
| `metadata.labels` | Tag `app=welcome`. A Service can find this Pod through it |
| `spec.containers` | List of containers in the Pod (the `-` starts a list item) |
| `name` | Container name inside the Pod |
| `image` | Image to pull. `kicbase/echo-server:1.0` is a tiny web server that echoes the request |
| `containerPort: 8080` | Documents that the app listens on 8080 |

**Important:** `containerPort` is documentation. It does not open or change anything. The real port is whatever the app listens on. This echo server listens on **8080**.

**Weakness of a bare Pod:** if it is deleted or its node dies, nothing recreates it. That is why we use a Deployment.

### 3.2 `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: welcome-deployment
  labels:
    app: welcome
spec:
  replicas: 2
  selector:
    matchLabels:
      app: welcome
  template:
    metadata:
      labels:
        app: welcome
    spec:
      containers:
        - name: welcome-container
          image: kicbase/echo-server:1.0
          ports:
            - containerPort: 8080     # note the space after the colon
```

| Section | Meaning |
|---|---|
| `apiVersion: apps/v1` | Deployments live in the `apps` API group |
| `replicas: 2` | Always keep 2 Pods running |
| `selector.matchLabels` | Which Pods this Deployment owns: those with `app=welcome` |
| `template` | The blueprint for each Pod. Everything under it is a Pod spec |
| `template.metadata.labels` | Labels stamped on each Pod. **Must match `selector`**, otherwise the Deployment is rejected |

What a Deployment creates behind the scenes:

```
Deployment → ReplicaSet → Pod (welcome-deployment-7d9f8b6c5-abcde)
                       → Pod (welcome-deployment-7d9f8b6c5-fghij)
```

The Pod names are generated: `<deployment>-<replicaset-hash>-<random>`.

What you get compared to a bare Pod:

- **Self-healing:** delete a Pod and a new one appears.
- **Scaling:** change `replicas`.
- **Rolling updates and rollback:** change the image and Kubernetes replaces Pods gradually.

### 3.3 `service-nodeport.yaml`

```yaml
apiVersion: v1
kind: Service              # capital S, kinds are case sensitive
metadata:
  name: welcome-service
spec:
  type: NodePort
  selector:
    app: welcome
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

| Field | Meaning |
|---|---|
| `selector: app: welcome` | Send traffic to Pods that have this label (and are Ready) |
| `port: 8080` | Port of the Service itself, inside the cluster (`welcome-service:8080`) |
| `targetPort: 8080` | Port on the Pod/container where the app really listens |
| `nodePort: 30080` | Port opened on every node. Must be in 30000-32767. Omit it to get a random one |

**The port chain (very important):**

```
Outside → NodeIP:30080 → Service port 8080 → Pod targetPort 8080 → app listening on 8080
          (nodePort)       (port)             (targetPort)           (containerPort)
```

### 3.4 `loadbalancer.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: welcome-lb
spec:
  type: LoadBalancer
  selector:
    app: welcome
  ports:
    - port: 80
      targetPort: 8080
```

```
Outside → LB :80 → Service port 80 → Pod targetPort 8080 → app on 8080
```

- Users call port **80**, and the Pod keeps listening on **8080**. `targetPort` does the translation.
- On a cloud provider (AWS, GCP) this creates a real load balancer with a public address.
- On minikube there is no cloud, so `EXTERNAL-IP` stays `<pending>` until you run `minikube tunnel`.

---

## 4. How everything depends on each other

| Relationship | Connected by | If it breaks |
|---|---|---|
| Deployment → Pods | `selector.matchLabels` = `template.labels` | The Deployment is rejected |
| Service → Pods | `Service.selector` = Pod labels | Service has **no endpoints**, no traffic |
| Service → app | `targetPort` = port the app listens on | Connection refused |
| Pod → image | `image` name and tag | `ImagePullBackOff` |

Check the Service link at any time:

```bash
kubectl get endpoints welcome-service
```

- Shows Pod IPs: connected.
- Shows `<none>`: label mismatch, or Pods not Ready.

---

## 5. Step-by-step run

### Step 0: Start the cluster

```bash
minikube start --driver=docker
kubectl get nodes
```

### Step 1: Deploy the app (Deployment creates the Pods automatically)

```bash
kubectl apply -f deployment.yaml
kubectl get deployment
kubectl get rs
kubectl get pods -o wide
```

You will see 2 Pods named `welcome-deployment-xxxxx-yyyyy`. You only applied the Deployment, and it created the ReplicaSet and the Pods for you.

### Step 2: Create the NodePort Service

```bash
kubectl apply -f service-nodeport.yaml
kubectl get svc
kubectl get endpoints welcome-service     # should list 2 Pod IPs
```

### Step 3: Open it in the browser

```bash
minikube service welcome-service          # opens the URL in your browser
minikube service welcome-service --url    # only prints the URL
```

Why `minikube service`? With the Docker driver on macOS or Windows, the node IP is not directly reachable from your machine. This command creates a tunnel. On Linux you can also use `curl http://$(minikube ip):30080`.

### Step 4: Inspect

```bash
kubectl describe pod <pod-name>           # read the Events section at the bottom
kubectl describe svc welcome-service
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- sh
```

### Step 5: Try the LoadBalancer

```bash
kubectl apply -f loadbalancer.yaml
kubectl get svc                           # EXTERNAL-IP shows <pending>
minikube tunnel                           # run in a second terminal, keep it open
kubectl get svc                           # EXTERNAL-IP now has a value
curl http://<EXTERNAL-IP>
```

### Step 6: Cleanup

```bash
kubectl delete -f loadbalancer.yaml
kubectl delete -f service-nodeport.yaml
kubectl delete -f deployment.yaml
```

Or delete everything with the label:

```bash
kubectl delete all -l app=welcome
```

---

## 6. Experiments (this is where the learning happens)

### A. Self-healing

```bash
kubectl get pods
kubectl delete pod <one-pod-name>
kubectl get pods                          # a new Pod is already being created
```

### B. Scaling

```bash
kubectl scale deployment welcome-deployment --replicas=4
kubectl get endpoints welcome-service     # 4 IPs now
kubectl scale deployment welcome-deployment --replicas=2
```

### C. Load balancing

Call the service several times. The echo server prints its hostname, so you can see different Pods answer:

```bash
curl $(minikube service welcome-service --url)
```

### D. Break the label link on purpose

```bash
kubectl patch svc welcome-service -p '{"spec":{"selector":{"app":"wrong"}}}'
kubectl get endpoints welcome-service     # <none>, the app is unreachable
kubectl apply -f service-nodeport.yaml    # restore
```

### E. The bare Pod is also picked up

`pod.yaml` also has `app: welcome`, so the Service treats it as a backend too:

```bash
kubectl apply -f pod.yaml
kubectl get endpoints welcome-service     # 3 IPs: 2 from the Deployment, 1 from the bare Pod
kubectl delete pod welcome-pod            # deleted for good, nobody recreates it
```

The Service does not care who created a Pod. It only looks at labels.

### F. Rolling update and rollback

```bash
kubectl set image deployment/welcome-deployment welcome-container=kicbase/echo-server:1.0
kubectl rollout status deployment/welcome-deployment
kubectl rollout history deployment/welcome-deployment
kubectl rollout undo deployment/welcome-deployment
```

---

## 7. Common errors

| Error | Cause | Fix |
|---|---|---|
| `no matches for kind "service"` | `kind` is case sensitive | Write `kind: Service` |
| `ports[0]: expected map, got string` or similar | `containerPort:8080` has no space after the colon | Write `containerPort: 8080` |
| `kubestl: command not found` | Typo | `kubectl` |
| `kubectl describe pod pode-name` not found | Placeholder typed literally | Use a real name from `kubectl get pods` |
| Service has `<none>` endpoints | Label mismatch or Pods not Ready | Compare `kubectl get pods --show-labels` with the Service selector |
| `EXTERNAL-IP` stuck on `<pending>` | minikube has no cloud load balancer | Run `minikube tunnel` |
| Browser cannot reach `NodeIP:30080` | Docker driver on macOS/Windows | Use `minikube service welcome-service` |
| `connection refused` | `targetPort` does not match the app port | App listens on 8080, so `targetPort: 8080` |
| `ImagePullBackOff` | Wrong image name or tag | Check the `image:` line |

---

## 8. Command cheat sheet

```bash
kubectl apply -f <file>.yaml          # create or update
kubectl delete -f <file>.yaml         # delete what the file created
kubectl get pods|deploy|rs|svc|endpoints
kubectl get pods --show-labels
kubectl get pods -l app=welcome
kubectl describe <type> <name>
kubectl logs <pod>
kubectl exec -it <pod> -- sh
kubectl scale deployment <name> --replicas=N
kubectl rollout status|history|undo deployment/<name>
minikube service <svc-name>
minikube tunnel
minikube dashboard
```

---

## 9. Review questions

1. Which field links a Service to its Pods?
2. What is the difference between `port`, `targetPort`, `nodePort`, and `containerPort`?
3. Why does deleting a Pod from a Deployment bring it back but not a bare Pod?
4. What does a NodePort Service create automatically besides the node port?
5. Why is `EXTERNAL-IP` `<pending>` on minikube, and how do you fix it?
6. What happens if the Deployment `selector` does not match the `template` labels?
7. If you apply both `pod.yaml` and `deployment.yaml`, how many Pods does the Service route to?

<details>
<summary>Answers</summary>

1. The Service `selector` matching the Pod labels
2. `port`: Service port. `targetPort`: Pod port the app listens on. `nodePort`: port opened on every node. `containerPort`: documentation of the app's port
3. The ReplicaSet created by the Deployment keeps the Pod count equal to `replicas`
4. A ClusterIP
5. No cloud load balancer exists. Run `minikube tunnel`
6. The API server rejects the Deployment
7. Three: 2 from the Deployment plus the bare Pod, because all have `app=welcome`

</details>

---

