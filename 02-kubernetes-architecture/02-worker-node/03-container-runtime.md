# Container Runtime

> Status: 📘 Notes ready

## Idea

The **container runtime** is the software on each node that actually pulls images and runs containers. The kubelet does not run containers itself. It asks the runtime.

## CRI: the plug between them

Kubernetes talks to runtimes through the **CRI (Container Runtime Interface)**, a gRPC API. Any runtime that speaks CRI works.

```
kubelet ──CRI (gRPC)──► containerd ──► runc ──► Linux namespaces + cgroups ──► your container
                        (high-level)   (low-level OCI runtime)
```

| Layer | Role |
|---|---|
| **kubelet** | Decides what should run |
| **CRI runtime** (containerd, CRI-O) | Pulls images, manages container lifecycle, sets up Pod sandboxes |
| **OCI runtime** (runc) | Creates the container process using namespaces and cgroups |

## Which runtime?

| Runtime | Notes |
|---|---|
| **containerd** | Most common. Default on EKS, GKE, AKS and kind |
| **CRI-O** | Built for Kubernetes, used by OpenShift |
| **Docker Engine** | Not a CRI runtime. Kubernetes removed its built-in support (dockershim) in **v1.24**. An adapter, `cri-dockerd`, exists |

**Your Docker images still work.** Images follow the **OCI image standard**, so anything built with `docker build` runs on containerd. Only the *runtime on the node* changed.

## The Pod sandbox ("pause" container)

For every Pod the runtime first creates a **sandbox**: a tiny `pause` container that holds the Pod's **network namespace**. Your containers then join it. This is why containers in a Pod share one IP and talk over `localhost`.

```
Pod
├── pause container      ← owns the IP and network namespace
├── app container   ─┐
└── sidecar         ─┴── join the pause container's namespaces
```

## Images and pulling

```yaml
containers:
  - name: web
    image: nginx:1.27
    imagePullPolicy: IfNotPresent    # Always | IfNotPresent | Never
```

| Policy | Behavior | Default when |
|---|---|---|
| `IfNotPresent` | Pull only if the node does not have it | Tag is specific (`nginx:1.27`) |
| `Always` | Check the registry every time | Tag is `:latest` or missing |
| `Never` | Never pull | Never by default |

Private registries need `imagePullSecrets`. Pulls are done by the **runtime on the node**, so `ImagePullBackOff` is a runtime or registry problem.

## Debugging on the node: `crictl`

`crictl` talks CRI directly, so it shows what Kubernetes sees:

```bash
sudo crictl ps                  # running containers
sudo crictl ps -a               # including exited
sudo crictl pods                # Pod sandboxes
sudo crictl images              # images on the node
sudo crictl logs <container-id>
sudo crictl inspect <container-id>
```

## Lab

```bash
# 1. Which runtime is each node using?
kubectl get nodes -o wide                    # CONTAINER-RUNTIME column, for example containerd://1.7.x

# 2. Run a Pod and find it on the node
kubectl run rt-demo --image=nginx:1.27
kubectl get pod rt-demo -o wide              # note the NODE
minikube ssh                                 # or: docker exec -it <kind-node> bash
sudo crictl pods | grep rt-demo              # the sandbox
sudo crictl ps | grep nginx                  # the container
sudo crictl images | grep nginx
exit

# 3. The pull policy in action
kubectl get pod rt-demo -o jsonpath='{.spec.containers[0].imagePullPolicy}{"\n"}'

# 4. Break the pull on purpose
kubectl run bad-image --image=nginx:does-not-exist
kubectl describe pod bad-image | tail -8     # Failed to pull image ... ErrImagePull / ImagePullBackOff

# Cleanup
kubectl delete pod rt-demo bad-image
```

To try another runtime on minikube: `minikube start -p rt --container-runtime=containerd`.

## Gotchas

- "Kubernetes dropped Docker" means the **runtime on nodes**, not Docker images or `docker build`.
- `docker ps` on a modern node may show nothing. Use `crictl ps`.
- Containers in a Pod share the **sandbox**, so killing the pause container restarts the whole Pod.

## Check yourself

1. What is the CRI?
2. Name two CRI runtimes.
3. Do images built with Docker still run on containerd?
4. What does the pause container do?
5. Which tool lists containers on a node at the CRI level?

<details>
<summary>Answers</summary>

1. The interface (gRPC API) the kubelet uses to talk to a container runtime
2. containerd and CRI-O
3. Yes. Both follow the OCI image standard
4. Holds the Pod's network namespace (and IP) so all containers in the Pod share it
5. `crictl`

</details>