# Kubernetes Learning Notes

A structured, hands-on path: **Architecture → Components → Step-by-step labs**.
Use this repo for review. Every section has commands and YAML you can run.

---

## Table of Contents

1. [What is Kubernetes and why it exists](#1-what-is-kubernetes-and-why-it-exists)
2. [Architecture](#2-architecture)
3. [Core Components and Objects](#3-core-components-and-objects)
4. [Lab Setup](#4-lab-setup)
5. [Step-by-Step Learning Path](#5-step-by-step-learning-path)
6. [kubectl Cheat Sheet](#6-kubectl-cheat-sheet)
7. [Debugging Playbook](#7-debugging-playbook)
8. [Practice Projects and Roadmap](#8-practice-projects-and-roadmap)

---

## 1. What is Kubernetes and why it exists

Docker runs a container on **one machine**. Real systems need:

| Problem | Kubernetes answer |
|---|---|
| Container crashes | Restarts it automatically (self-healing) |
| Traffic grows | Scales replicas up/down (HPA) |
| New version release | Rolling updates and rollbacks |
| Many containers need to find each other | Service discovery + DNS |
| Config and secrets differ per environment | ConfigMaps and Secrets |
| Run across many servers | Scheduler places Pods on nodes |

**Core idea: declarative desired state.**
You write YAML saying *what you want* ("3 copies of my app"). Kubernetes continuously
works to make reality match it. This loop is called **reconciliation**.

```
Desired state (YAML)  ──►  Controller compares  ◄──  Actual state (cluster)
                                  │
                         fixes any difference
```

---

## 2. Architecture

A **cluster** = **Control Plane** (the brain) + **Worker Nodes** (the muscle).

```
                        ┌───────────────── CONTROL PLANE ─────────────────┐
  kubectl / CI/CD ────► │  kube-apiserver  ◄──►  etcd (cluster database)   │
                        │       ▲                                          │
                        │       ├── kube-scheduler (picks a node for Pods) │
                        │       ├── kube-controller-manager (reconcilers)  │
                        │       └── cloud-controller-manager (AWS/GCP/...) │
                        └──────────────────────────────────────────────────┘
                                          │
              ┌───────────────────────────┴───────────────────────────┐
     ┌────────▼─────────┐                                   ┌─────────▼────────┐
     │   WORKER NODE 1  │                                   │   WORKER NODE 2  │
     │  kubelet         │                                   │  kubelet         │
     │  kube-proxy      │                                   │  kube-proxy      │
     │  container runtime│                                  │  container runtime│
     │  [Pod] [Pod]     │                                   │  [Pod] [Pod]     │
     └──────────────────┘                                   └──────────────────┘
```

### 2.1 Control Plane components

| Component | Role | Remember it as |
|---|---|---|
| **kube-apiserver** | The only front door. All requests (kubectl, kubelet, controllers) go through it. Validates, authenticates, authorizes, then stores. | Receptionist |
| **etcd** | Distributed key-value store holding the entire cluster state. Back it up! | Database / source of truth |
| **kube-scheduler** | Watches for unscheduled Pods and chooses a node (based on resources, affinity, taints). | Matchmaker |
| **kube-controller-manager** | Runs many control loops: Node, ReplicaSet, Deployment, Job, Endpoints controllers. | Thermostat: keeps reality equal to desired |
| **cloud-controller-manager** | Talks to cloud APIs (create load balancers, volumes, node lifecycle). | Cloud adapter |

### 2.2 Worker Node components

| Component | Role |
|---|---|
| **kubelet** | Agent on each node. Gets Pod specs from the API server, tells the runtime to run containers, reports health/status. |
| **kube-proxy** | Maintains network rules (iptables/IPVS) so Services route traffic to the right Pods. |
| **Container runtime** | Actually runs containers (containerd, CRI-O). Docker is not required anymore. |

### 2.3 What happens when you run `kubectl apply -f deployment.yaml`

1. `kubectl` sends the YAML to **kube-apiserver** (authN → authZ → admission).
2. API server stores the Deployment in **etcd**.
3. **Deployment controller** sees it and creates a **ReplicaSet**.
4. **ReplicaSet controller** creates the **Pod** objects.
5. **Scheduler** notices Pods with no node and assigns each one.
6. **kubelet** on the chosen node sees its Pod, asks the runtime to pull the image and start containers.
7. kubelet reports status back to the API server. `kubectl get pods` shows `Running`.

Every component only talks to the API server, never directly to each other. That is the key design.

### 2.4 Networking model (4 rules)

1. Every Pod gets its own IP.
2. Pods can reach all other Pods without NAT.
3. Nodes can reach all Pods.
4. Services give a stable virtual IP/DNS name in front of changing Pod IPs.

Implemented by a **CNI plugin** (Calico, Cilium, Flannel, AWS VPC CNI on EKS).

---

## 3. Core Components and Objects

### 3.1 Object map

```
Deployment ──manages──► ReplicaSet ──manages──► Pods ──contain──► Containers
                                                  ▲
Service (stable IP/DNS) ──selects by labels───────┘
Ingress ──routes HTTP(S) to──► Services
ConfigMap / Secret ──injected into──► Pods (env vars or files)
PVC ──claims──► PV (storage) ──mounted in──► Pods
```

### 3.2 Quick definitions

| Object | What it is | When to use |
|---|---|---|
| **Pod** | Smallest unit. 1+ containers sharing network and storage. | Never create directly in production; use a controller |
| **ReplicaSet** | Keeps N identical Pods running | Created by Deployment automatically |
| **Deployment** | Declarative updates, rollout, rollback for stateless apps | APIs, web apps (most common) |
| **StatefulSet** | Pods with stable names and storage (`db-0`, `db-1`) | Databases, Kafka |
| **DaemonSet** | One Pod per node | Log agents, monitoring |
| **Job / CronJob** | Run-to-completion / scheduled tasks | Batch, backups |
| **Service** | Stable network endpoint for a set of Pods | Always, for networking |
| **Ingress** | HTTP/HTTPS routing, TLS, host/path rules | Expose multiple apps on one LB |
| **ConfigMap** | Non-secret config | Env settings |
| **Secret** | Sensitive data (base64, not encrypted by default) | Passwords, tokens |
| **Namespace** | Virtual cluster for isolation | dev / staging / prod |
| **PV / PVC / StorageClass** | Persistent storage | Data that outlives Pods |
| **HPA** | Autoscale Pods by CPU/memory/metrics | Variable load |
| **RBAC** | Who can do what | Security |

### 3.3 Service types

| Type | Reachable from | Use |
|---|---|---|
| `ClusterIP` (default) | Inside cluster only | Internal service-to-service |
| `NodePort` | `<NodeIP>:30000-32767` | Dev/testing |
| `LoadBalancer` | Cloud LB external IP | Production external access |
| `ExternalName` | DNS alias to external host | Point to outside service |

---

## 4. Lab Setup

Pick one local cluster tool:

```bash
# Option A: minikube (easy, single node)
minikube start --driver=docker
minikube status

# Option B: kind (Kubernetes IN Docker, great for multi-node)
kind create cluster --name learn
```

Multi-node kind cluster (`kind-config.yaml`):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name learn --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
```

Install `kubectl` and verify:

```bash
kubectl version --client
kubectl config get-contexts
```

---

## 5. Step-by-Step Learning Path

### Step 1: Explore the cluster

```bash
kubectl get nodes -o wide
kubectl get pods -A                       # all namespaces
kubectl get pods -n kube-system           # see control plane components
kubectl describe node <node-name>
```

Goal: spot `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `kube-proxy`, `coredns`.

### Step 2: Your first Pod

`pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    app: hello
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod hello-pod
kubectl logs hello-pod
kubectl exec -it hello-pod -- sh
kubectl port-forward pod/hello-pod 8080:80   # open http://localhost:8080
kubectl delete -f pod.yaml
```

Observe: delete the Pod and it stays gone. A bare Pod has no self-healing. That is why we need Deployments.

**YAML anatomy** (every object has these 4 top-level keys):

```yaml
apiVersion: ...   # which API group/version
kind: ...         # object type
metadata: ...     # name, namespace, labels, annotations
spec: ...         # desired state
```

### Step 3: Deployments (self-healing, scaling, rollouts)

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web          # must match selector above
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods

# Self-healing test
kubectl delete pod <one-pod-name>      # a new one appears instantly

# Scale
kubectl scale deployment web --replicas=5

# Rolling update
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout history deployment/web

# Rollback
kubectl rollout undo deployment/web
```

### Step 4: Services (stable networking)

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: ClusterIP
  selector:
    app: web              # routes to Pods with this label
  ports:
    - port: 80            # Service port
      targetPort: 80      # container port
```

```bash
kubectl apply -f service.yaml
kubectl get svc
kubectl get endpoints web-svc          # the Pod IPs behind the Service

# Test from inside the cluster (DNS: <svc>.<namespace>.svc.cluster.local)
kubectl run tmp --rm -it --image=busybox:1.36 -- wget -qO- http://web-svc

# Test from your laptop
kubectl port-forward svc/web-svc 8080:80
```

Change `type: NodePort` or `LoadBalancer` to expose externally. On minikube: `minikube service web-svc`.

### Step 5: Namespaces

```bash
kubectl create namespace dev
kubectl apply -f deployment.yaml -n dev
kubectl get pods -n dev
kubectl config set-context --current --namespace=dev   # change default namespace
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

### Step 6: ConfigMaps and Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  API_URL: "https://api.example.com"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:                 # plain text here, stored base64-encoded
  DB_PASSWORD: "change-me"
```

Use them in a Pod/Deployment:

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      envFrom:
        - configMapRef:
            name: app-config
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
```

Or mount as files:

```yaml
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

```bash
kubectl create secret generic db-creds --from-literal=password=abc123
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

> Secrets are only base64-encoded by default. In production enable encryption at rest
> and use AWS Secrets Manager / External Secrets Operator.

### Step 7: Health probes and resource limits

```yaml
    spec:
      containers:
        - name: app
          image: my-app:1.0
          resources:
            requests:            # used by scheduler to place the Pod
              cpu: "100m"
              memory: "128Mi"
            limits:              # hard ceiling
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:        # ready to receive traffic?
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:         # alive? if not, restart
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
```

| Probe | Failure result |
|---|---|
| readiness | Pod removed from Service endpoints (no restart) |
| liveness | Container restarted |
| startup | Delays other probes for slow-starting apps |

Exceeding the memory limit gives `OOMKilled`. Exceeding CPU limit means throttling.

### Step 8: Ingress (HTTP routing)

Enable an ingress controller first:

```bash
minikube addons enable ingress
# or install ingress-nginx on kind / real clusters
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

Add `myapp.local` to `/etc/hosts` pointing at the ingress IP to test.

### Step 9: Persistent storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo hello > /data/file.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
kubectl get pvc,pv
```

Flow: **StorageClass** provisions a **PV** dynamically when a **PVC** asks for it.

### Step 10: StatefulSet, DaemonSet, Job, CronJob

```yaml
# Job: run once to completion
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: job
          image: busybox:1.36
          command: ["sh", "-c", "echo done"]
---
# CronJob: every day at 02:00
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: task
              image: busybox:1.36
              command: ["sh", "-c", "echo backup"]
```

StatefulSet gives stable identity (`db-0`, `db-1`) and per-Pod storage via `volumeClaimTemplates`.
DaemonSet runs one Pod on every node (log shippers, node exporters).

### Step 11: Autoscaling (HPA)

Needs metrics-server (`minikube addons enable metrics-server`).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```bash
kubectl get hpa -w
kubectl top pods
```

The Deployment must define CPU `requests`, otherwise HPA cannot calculate utilization.

### Step 12: Scheduling controls

```yaml
spec:
  nodeSelector:
    disktype: ssd
  affinity:
    podAntiAffinity:                 # spread replicas across nodes
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

```bash
kubectl label node <node> disktype=ssd
kubectl taint nodes <node> dedicated=gpu:NoSchedule
```

Taints repel Pods; tolerations let specific Pods in. Affinity attracts Pods.

### Step 13: RBAC (security basics)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
  - kind: ServiceAccount
    name: app-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl auth can-i list pods --as=system:serviceaccount:default:app-sa
```

Also learn: NetworkPolicy, Pod Security Standards, `securityContext` (`runAsNonRoot`, `readOnlyRootFilesystem`).

### Step 14: Helm and Kustomize

```bash
# Helm: package manager
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-redis bitnami/redis
helm list
helm uninstall my-redis

# Kustomize: built into kubectl, overlays per environment
kubectl apply -k overlays/prod
```

### Step 15: Real cluster (EKS) and production topics

- EKS with Terraform, managed node groups or Fargate
- AWS Load Balancer Controller for Ingress/ALB
- IAM Roles for Service Accounts (IRSA) so Pods get AWS permissions
- GitOps with Argo CD or Flux
- Observability: Prometheus + Grafana, Loki, OpenTelemetry
- Cluster Autoscaler / Karpenter
- Backup etcd / Velero, upgrade strategy, PodDisruptionBudgets

---

## 6. kubectl Cheat Sheet

```bash
# Context and config
kubectl config get-contexts
kubectl config use-context <name>

# Create / update / delete
kubectl apply -f file.yaml
kubectl apply -f ./folder/
kubectl delete -f file.yaml
kubectl delete pod <name>

# Inspect
kubectl get pods|deploy|svc|ing|pvc|nodes
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods --show-labels
kubectl get pods -l app=web
kubectl get all -n <ns>
kubectl describe <type> <name>

# Logs and shell
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs -f <pod>
kubectl logs <pod> --previous           # after a crash
kubectl exec -it <pod> -- sh

# Networking
kubectl port-forward svc/<name> 8080:80
kubectl get endpoints <svc>

# Rollouts
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>
kubectl rollout undo deploy/<name>
kubectl rollout restart deploy/<name>

# Scaling
kubectl scale deploy/<name> --replicas=5
kubectl autoscale deploy/<name> --min=2 --max=10 --cpu-percent=70

# Resource usage
kubectl top nodes
kubectl top pods

# Generate YAML fast (no cluster change)
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run test --image=nginx --dry-run=client -o yaml

# Docs inside the terminal
kubectl explain pod.spec.containers
kubectl api-resources

# Events (very useful)
kubectl get events --sort-by=.metadata.creationTimestamp
```

Helpful aliases:

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kaf='kubectl apply -f'
```

---

## 7. Debugging Playbook

| Status | Meaning | Check |
|---|---|---|
| `Pending` | Not scheduled | `kubectl describe pod` → events: insufficient CPU/memory, taints, unbound PVC |
| `ImagePullBackOff` / `ErrImagePull` | Cannot pull image | Wrong name/tag, private registry needs `imagePullSecrets` |
| `CrashLoopBackOff` | Container keeps crashing | `kubectl logs <pod> --previous`, bad env/config, failing probe |
| `OOMKilled` | Exceeded memory limit | Raise limit or fix memory leak |
| `CreateContainerConfigError` | Missing ConfigMap/Secret | Check names and keys |
| `Running` but not working | App or networking issue | Check readiness, Service selector, endpoints |

Debug flow:

```bash
kubectl get pods
kubectl describe pod <pod>              # read the Events section at the bottom
kubectl logs <pod> [--previous]
kubectl exec -it <pod> -- sh
kubectl get svc,endpoints               # empty endpoints = selector/label mismatch or not ready
kubectl run dbg --rm -it --image=busybox:1.36 -- sh    # test DNS / connectivity
```

Service not reachable? Checklist:
1. Do Service `selector` labels match Pod labels?
2. Does `targetPort` match the container port?
3. Are Pods `Ready`? (`kubectl get endpoints`)
4. Does DNS resolve? (`nslookup web-svc` from a debug Pod)

---

## 8. Practice Projects and Roadmap

### Suggested order

- [ ] Week 1: Architecture, local cluster, Pods, Deployments, Services
- [ ] Week 2: ConfigMaps, Secrets, probes, resources, Namespaces
- [ ] Week 3: Ingress, PV/PVC, StatefulSet, Jobs/CronJobs
- [ ] Week 4: HPA, scheduling, RBAC, Helm/Kustomize
- [ ] Week 5+: EKS + Terraform, CI/CD (GitLab to cluster), GitOps, monitoring

### Hands-on projects

1. Containerize a Node.js/NestJS API, deploy with 3 replicas, Service + Ingress
2. Add a database (StatefulSet + PVC) and wire credentials through Secrets
3. Add readiness/liveness probes, limits, and an HPA; load test with `hey` or `k6`
4. Build a GitLab CI pipeline: build image, push to registry, `kubectl set image` or Helm upgrade
5. Provision an EKS cluster with Terraform and deploy the same app

### Certification path

- **CKAD** (developer focus), **CKA** (admin), **CKS** (security)
- Practice: killercoda.com, killer.sh, `kubectl explain` and `--dry-run=client -o yaml` for speed

### Useful references

- Official docs: https://kubernetes.io/docs/
- kubectl cheat sheet: https://kubernetes.io/docs/reference/kubectl/quick-reference/
- Kubernetes the Hard Way (deep understanding): https://github.com/kelseyhightower/kubernetes-the-hard-way

---

## Quick Review Questions

1. Which component is the only one that talks to etcd?
2. What is the difference between a Deployment and a ReplicaSet?
3. Why do Pod IPs change and how does a Service solve it?
4. Difference between readiness and liveness probes?
5. What happens, step by step, after `kubectl apply -f deployment.yaml`?
6. When would you use a StatefulSet instead of a Deployment?
7. What is the difference between resource `requests` and `limits`?
8. How does a taint differ from node affinity?

<details>
<summary>Answers</summary>

1. kube-apiserver
2. Deployment manages ReplicaSets and provides rollout/rollback; ReplicaSet only keeps N Pods running
3. Pods are ephemeral; a Service gives a stable virtual IP/DNS and load-balances to ready Pods via label selector
4. Readiness controls traffic routing; liveness triggers container restart
5. See section 2.3
6. Stable identity and storage (databases, brokers)
7. Requests are used for scheduling; limits are the enforced ceiling
8. Taints repel Pods from nodes unless tolerated; affinity attracts Pods to nodes or other Pods

</details>