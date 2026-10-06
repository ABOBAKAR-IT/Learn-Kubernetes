# kubectl

> Status: 📘 Notes ready

`kubectl` is the command line client for the Kubernetes API. Every command becomes an API call to the API server.

## Syntax

```
kubectl <verb> <resource> [name] [flags]
         │        │         │      └── -n namespace, -o format, -l selector ...
         │        │         └── optional: one specific object
         │        └── pod, deployment, service ...
         └── get, describe, apply, delete ...
```

## The verbs you need

| Verb | Does |
|---|---|
| `get` | List objects |
| `describe` | Human-readable details and **Events** |
| `apply` | Create or update from a file (declarative) |
| `create` | Create (fails if it exists) |
| `delete` | Delete |
| `edit` | Open the live object in your editor |
| `patch` | Change a field with a small JSON snippet |
| `logs` | Container output |
| `exec` | Run a command inside a container |
| `port-forward` | Tunnel a local port to a Pod or Service |
| `scale` | Change replicas |
| `rollout` | Status, history, undo for Deployments |
| `label` / `annotate` | Change labels and annotations |
| `top` | CPU and memory (needs metrics-server) |
| `explain` | Built-in documentation of any field |

## Resources and short names

| Resource | Short | Resource | Short |
|---|---|---|---|
| pods | `po` | namespaces | `ns` |
| deployments | `deploy` | configmaps | `cm` |
| replicasets | `rs` | persistentvolumes | `pv` |
| daemonsets | `ds` | persistentvolumeclaims | `pvc` |
| services | `svc` | serviceaccounts | `sa` |
| nodes | `no` | ingresses | `ing` |
| statefulsets | `sts` | horizontalpodautoscalers | `hpa` |

```bash
kubectl api-resources            # every resource type, short names, namespaced or not
```

## Namespaces and context

```bash
kubectl get pods -n dev                              # one namespace
kubectl get pods -A                                  # all namespaces
kubectl config set-context --current --namespace=dev # change the default
kubectl config current-context
```

## Output formats (`-o`)

```bash
kubectl get pods -o wide                             # extra columns (node, IP)
kubectl get pod web -o yaml                          # full object
kubectl get pod web -o json
kubectl get pods -o name                             # just pod/web-abc
kubectl get pod web -o jsonpath='{.status.podIP}'
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods -w                                  # watch changes live
```

## Filtering

```bash
kubectl get pods -l app=web                          # by label
kubectl get pods -l 'env in (dev,qa)'
kubectl get pods --show-labels
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=worker1
```

## Imperative vs declarative

| | Imperative | Declarative |
|---|---|---|
| Example | `kubectl run`, `kubectl create deployment`, `kubectl scale` | `kubectl apply -f file.yaml` |
| You say | "do this now" | "make it look like this file" |
| Best for | Quick tests, generating YAML | Real work, Git, repeatable |

```bash
kubectl create -f pod.yaml          # fails if it already exists
kubectl apply  -f pod.yaml          # create or update (idempotent, preferred)
kubectl apply  -f ./manifests/      # a whole folder
kubectl delete -f pod.yaml          # delete what the file defines
```

## The YAML generator trick (huge time saver)

```bash
kubectl run web --image=nginx:1.27 --port=80 --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > deploy.yaml
kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml
```

`--dry-run=client` prints what would be created without sending it. `--dry-run=server` asks the API server to validate it for real, still without saving.

```bash
kubectl apply -f deploy.yaml --dry-run=server     # validate against the real API
kubectl diff -f deploy.yaml                       # what would change vs the live object
```

## Built-in documentation

```bash
kubectl explain pod
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy --recursive
```

Use `explain` whenever you forget a field name. It is available offline in the terminal (and in the CKA/CKAD exams).

## Debugging commands

```bash
kubectl describe pod web                              # read the Events at the bottom
kubectl logs web                                      # current container
kubectl logs web --previous                           # the container that just crashed
kubectl logs -f web --tail=50 --since=10m             # follow, last 50 lines, last 10 min
kubectl logs web -c sidecar                           # a specific container in the Pod
kubectl logs -l app=web --all-containers              # all Pods with a label
kubectl exec -it web -- sh                            # shell inside
kubectl exec web -- env                               # run one command
kubectl port-forward pod/web 8080:80                  # localhost:8080 -> Pod port 80
kubectl port-forward svc/web 8080:80                  # to a Service
kubectl cp web:/var/log/app.log ./app.log             # copy a file out
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl wait --for=condition=Ready pod/web --timeout=60s
```

## Changing live objects

```bash
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.27
kubectl label pod web env=dev --overwrite
kubectl annotate deployment web kubernetes.io/change-cause="upgrade to 1.27"
kubectl patch deployment web -p '{"spec":{"replicas":3}}'
kubectl edit deployment web
kubectl rollout status deployment/web
kubectl rollout undo deployment/web
```

Live edits (`edit`, `patch`, `scale`) drift away from your YAML in Git. Prefer editing the file and running `apply`.

## Quality of life setup

```bash
# autocomplete (bash)
source <(kubectl completion bash)
# autocomplete (zsh)
source <(kubectl completion zsh)

# alias
alias k=kubectl
complete -o default -F __start_kubectl k          # bash: keep completion for the alias
```

Put these lines in `~/.bashrc` or `~/.zshrc`.

## Practice drill (10 minutes)

Do each from memory, then check the answer.

1. List all Pods in all namespaces.
2. Print only the names of the Pods in `kube-system`.
3. Generate a Deployment YAML for `nginx:1.27` with 3 replicas without creating it.
4. Show the field documentation for `pod.spec.containers.livenessProbe`.
5. Show which node each Pod runs on, using custom columns.
6. See what `apply` would change before applying.
7. Follow the logs of a Pod, last 20 lines only.
8. Get a shell in a running Pod.
9. Forward local port 8080 to a Service's port 80.
10. List only Pods labeled `app=web` and `env=dev`.

<details>
<summary>Answers</summary>

1. `kubectl get pods -A`
2. `kubectl get pods -n kube-system -o name`
3. `kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml`
4. `kubectl explain pod.spec.containers.livenessProbe`
5. `kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName`
6. `kubectl diff -f file.yaml`
7. `kubectl logs -f <pod> --tail=20`
8. `kubectl exec -it <pod> -- sh`
9. `kubectl port-forward svc/<name> 8080:80`
10. `kubectl get pods -l app=web,env=dev`

</details>

## Gotchas

- **Wrong context or namespace** is the number one mistake. Check `kubectl config current-context` and use `-n`.
- `kubectl get all` does **not** show everything (no ConfigMaps, Secrets, PVCs, Ingresses).
- `kubectl delete pod <name> --force --grace-period=0` skips graceful shutdown. Avoid it outside emergencies.
- Mixing `create` and `apply` on the same object can warn about a missing last-applied annotation. Pick `apply`.
- `--record` is deprecated. Use the `kubernetes.io/change-cause` annotation.

## Check yourself

1. What is the syntax pattern of a kubectl command?
2. What is the difference between `create` and `apply`?
3. How do you generate YAML without creating anything?
4. How do you read the logs of a container that just crashed?
5. How do you look up a field name inside the terminal?

<details>
<summary>Answers</summary>

1. `kubectl <verb> <resource> [name] [flags]`
2. `create` fails if the object exists. `apply` creates or updates and is idempotent
3. `--dry-run=client -o yaml`
4. `kubectl logs <pod> --previous`
5. `kubectl explain <resource>.<field>`

</details>