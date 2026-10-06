# Kubernetes Architecture (overview)

> Status: 📘 Notes ready
>
> This is the 5-minute overview. The deep dive, one file per component, is in [`../02-kubernetes-architecture/`](../02-kubernetes-architecture/).

## The big picture

A **cluster** = a **control plane** (the brain) + **worker nodes** (the muscle).

```
                        ┌──────────────── CONTROL PLANE ────────────────┐
  kubectl / CI/CD ────► │  kube-apiserver  ◄──►  etcd (cluster database) │
                        │       ▲                                        │
                        │       ├── kube-scheduler (picks a node)        │
                        │       ├── kube-controller-manager (loops)      │
                        │       └── cloud-controller-manager (cloud API) │
                        └────────────────────────────────────────────────┘
                                          │
              ┌───────────────────────────┴──────────────────────────┐
     ┌────────▼─────────┐                                  ┌─────────▼────────┐
     │   WORKER NODE 1  │                                  │   WORKER NODE 2  │
     │ kubelet          │                                  │ kubelet          │
     │ kube-proxy       │                                  │ kube-proxy       │
     │ container runtime│                                  │ container runtime│
     │ [Pod] [Pod]      │                                  │ [Pod] [Pod]      │
     └──────────────────┘                                  └──────────────────┘
```

## Components in one line each

| Component | Where | One-line job |
|---|---|---|
| **kube-apiserver** | Control plane | The only front door. Everything talks to it |
| **etcd** | Control plane | Database holding all cluster state |
| **kube-scheduler** | Control plane | Chooses a node for each new Pod |
| **kube-controller-manager** | Control plane | Runs the reconciliation loops |
| **cloud-controller-manager** | Control plane | Talks to the cloud provider (load balancers, volumes) |
| **kubelet** | Each node | Starts and watches the Pods on its node |
| **kube-proxy** | Each node | Sets network rules so Services reach Pods |
| **Container runtime** | Each node | Runs the containers (containerd, CRI-O) |

## The key design rule

**Components never talk to each other directly. They all talk to the API server.**

## What happens on `kubectl apply -f deployment.yaml`

1. `kubectl` sends the YAML to the **API server** (authentication, authorization, admission checks).
2. The API server stores the Deployment in **etcd**.
3. The **Deployment controller** creates a **ReplicaSet**.
4. The **ReplicaSet controller** creates the **Pod** objects.
5. The **scheduler** assigns each unscheduled Pod to a node.
6. The **kubelet** on that node asks the runtime to pull the image and start the containers.
7. The kubelet reports status back. `kubectl get pods` shows `Running`.

## See it on your own cluster

```bash
kubectl get pods -n kube-system -o wide
# look for: kube-apiserver-*, etcd-*, kube-scheduler-*, kube-controller-manager-*, kube-proxy-*, coredns-*
```

## Check yourself

1. Which two parts make up a cluster?
2. Which component stores the cluster state?
3. Which component decides which node a Pod runs on?
4. Why do all components go through the API server?

<details>
<summary>Answers</summary>

1. Control plane and worker nodes
2. etcd (only the API server reads and writes it)
3. kube-scheduler
4. One front door means one place for authentication, authorization, validation and auditing, and components stay decoupled

</details>