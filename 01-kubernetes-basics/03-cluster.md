# Clusters, Contexts and kubeconfig

> Status: 📘 Notes ready

## What is a cluster?

A **cluster** is the set of machines (nodes) that Kubernetes manages as one system, plus the control plane that manages them. You talk to the cluster through its **API server**.

## Kinds of clusters

| Kind | Tools | Use it for |
|---|---|---|
| **Local** | minikube, kind, k3d, Docker Desktop | Learning, testing, CI |
| **Self-managed** | kubeadm, k3s, Rancher | Full control, on-premises |
| **Managed** | AWS **EKS**, Google GKE, Azure AKS | Production without running the control plane |

Learning path: local first (modules 00-14), then managed (module 15, EKS).

## Create a local cluster

```bash
# Option A: minikube (single node by default)
minikube start --driver=docker
minikube status

# Option B: kind (Kubernetes IN Docker), multi-node
cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF
kind create cluster --name learn --config kind-config.yaml
```

Verify:

```bash
kubectl cluster-info
kubectl get nodes
```

## kubeconfig: how kubectl knows which cluster to talk to

`kubectl` reads a file, by default `~/.kube/config`. It has three lists and one link between them:

```
clusters:  where is the API server?     (URL + certificate authority)
users:     who am I?                    (certificate, token, or exec plugin)
contexts:  cluster + user + namespace   ← a named shortcut
current-context: which context is active right now
```

```yaml
# shape of ~/.kube/config (simplified)
apiVersion: v1
kind: Config
clusters:
- name: kind-learn
  cluster:
    server: https://127.0.0.1:6443
users:
- name: kind-learn
  user: {}            # credentials live here
contexts:
- name: kind-learn
  context:
    cluster: kind-learn
    user: kind-learn
    namespace: default
current-context: kind-learn
```

### Context commands (you will use these daily)

```bash
kubectl config get-contexts                       # list all, * marks the active one
kubectl config current-context
kubectl config use-context kind-learn             # switch cluster
kubectl config view --minify                      # only the active context's config
kubectl config set-context --current --namespace=dev   # change default namespace
kubectl --context=minikube get nodes              # one-off, without switching
```

Tools that make this easier: `kubectx` and `kubens`.

### Several kubeconfig files

```bash
export KUBECONFIG=~/.kube/config:~/.kube/eks-config     # merged view
kubectl config get-contexts
```

## Lab

```bash
kind create cluster --name learn --config kind-config.yaml
minikube start -p mini                             # a second cluster
kubectl config get-contexts                        # two contexts now
kubectl config use-context kind-learn
kubectl get nodes                                  # 3 nodes
kubectl config use-context mini
kubectl get nodes                                  # 1 node
```

Cleanup:

```bash
kind delete cluster --name learn
minikube delete -p mini
```

## Gotchas

- **Always check the active context before deleting anything.** `kubectl config current-context` prevents "I deleted it on production".
- A kubeconfig contains **credentials**. Never commit it to Git (add `kubeconfig*` and `.kube/` to `.gitignore`).
- `kubectl` talking to the wrong cluster is the most common beginner surprise.

## Check yourself

1. What are the three parts a context combines?
2. How do you switch clusters?
3. Which file does `kubectl` read by default?
4. Why should a kubeconfig never be committed?

<details>
<summary>Answers</summary>

1. Cluster, user, and namespace
2. `kubectl config use-context <name>`
3. `~/.kube/config` (or the files listed in `$KUBECONFIG`)
4. It holds credentials that give access to the cluster

</details>