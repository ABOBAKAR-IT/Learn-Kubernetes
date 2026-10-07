# kubelet

> Status: 📘 Notes ready

## Idea

The **kubelet** is the agent on every node. It is the bridge between the control plane's decisions and real containers.

It runs as a **system service** on the node (not as a Pod, except in special setups like kind where the "node" is itself a container).

## What it does

| Job | Detail |
|---|---|
| **Registers the node** | Introduces the node to the API server with its capacity and labels |
| **Watches for its Pods** | Looks for Pods whose `nodeName` is this node |
| **Runs them** | Tells the container runtime (through CRI) to pull images and start containers |
| **Mounts volumes** | Prepares emptyDir, ConfigMap, Secret and PVC mounts for the Pod |
| **Runs probes** | Executes liveness, readiness and startup probes |
| **Restarts containers** | Applies `restartPolicy` |
| **Enforces limits** | Sets CPU and memory limits through Linux cgroups |
| **Reports status** | Sends Pod status and node conditions back to the API server |
| **Heartbeats** | Renews the node's Lease so the control plane knows it is alive |
| **Evicts Pods** | Kills Pods when the node runs low on memory or disk |
| **Cleans up** | Garbage collects unused images and dead containers |

```
API server ──"Pod web is yours"──► kubelet ──CRI──► container runtime ──► containers
     ▲                               │
     └────── status, probes, heartbeat ◄┘
```

## Pod lifecycle as the kubelet sees it

1. Create the Pod's **sandbox** (the network namespace, held by a tiny "pause" container).
2. Mount volumes.
3. Pull images and run **init containers** one by one.
4. Start the **app containers**.
5. Run **probes**, report `Ready`.
6. On exit or failure, restart according to `restartPolicy` (with back-off: 10s, 20s, 40s... up to 5 minutes, which you see as `CrashLoopBackOff`).

## Static Pods

A **static Pod** is defined by a file on the node, not through the API server. The kubelet watches a folder (default `/etc/kubernetes/manifests`) and keeps those Pods running by itself.

- This is how **kubeadm** runs `kube-apiserver`, `etcd`, `kube-scheduler` and `kube-controller-manager`. The kubelet starts the control plane before any API server exists.
- The kubelet creates a read-only **mirror Pod** in the API so you can see it. Its name ends with the node name, for example `etcd-minikube`.
- Deleting the mirror Pod with `kubectl` does **not** stop it. Delete the file instead.

## Where things live (kubeadm-style nodes)

| Item | Location |
|---|---|
| Service | `systemctl status kubelet` |
| Logs | `journalctl -u kubelet` |
| Config | `/var/lib/kubelet/config.yaml` |
| Static Pod folder | `/etc/kubernetes/manifests` |

## Lab

```bash
# Get a shell on a node
minikube ssh                         # minikube
# docker exec -it learn-worker bash  # kind (node name from: docker ps)

# On the node:
systemctl status kubelet --no-pager | head
ls /etc/kubernetes/manifests         # the control plane static Pods
sudo grep staticPodPath /var/lib/kubelet/config.yaml
sudo journalctl -u kubelet --no-pager | tail -5
```

Create a static Pod (still on the node):

```bash
sudo tee /etc/kubernetes/manifests/static-web.yaml > /dev/null <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  containers:
    - name: web
      image: nginx:1.27
EOF
exit
```

Back on your laptop:

```bash
kubectl get pods -o wide                 # static-web-<node-name>: the mirror Pod
kubectl delete pod static-web-<node-name>
kubectl get pods                         # it comes right back: the kubelet owns it
```

Remove it for real:

```bash
minikube ssh "sudo rm /etc/kubernetes/manifests/static-web.yaml"
kubectl get pods                         # gone
```

## Gotchas

- Node `NotReady` very often means the **kubelet is stopped or cannot reach the API server**. Check `systemctl status kubelet` and `journalctl -u kubelet`.
- Probes run in the kubelet, so a failing readiness probe only removes the Pod from Service endpoints, while a failing liveness probe restarts the container.
- The kubelet enforces limits, but the **scheduler** uses requests. They are different jobs.

## Check yourself

1. Where does the kubelet run?
2. What is a static Pod and who creates it?
3. Which component runs probes?
4. What does `CrashLoopBackOff` mean in kubelet terms?
5. Node is `NotReady`: first thing to check?

<details>
<summary>Answers</summary>

1. On every node, as a system service
2. A Pod defined by a manifest file on the node, run directly by the kubelet (no scheduler involved)
3. The kubelet
4. The container keeps failing and the kubelet is waiting an increasing back-off time between restarts
5. The kubelet service and its logs (`systemctl status kubelet`, `journalctl -u kubelet`)

</details>