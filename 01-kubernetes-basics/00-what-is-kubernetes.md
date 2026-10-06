# What is Kubernetes?

> Status: 📘 Notes ready

## The one-sentence answer

**Kubernetes (K8s) is an open-source platform that runs and manages containers across many machines, and keeps your applications running the way you described.**

- "K8s" = **K** + 8 letters + **s**.
- The name comes from the Greek word for *helmsman* (the person who steers a ship).
- Google open-sourced it in 2014, inspired by its internal system called Borg. It is now maintained by the **CNCF** (Cloud Native Computing Foundation).

## The problem it solves

Docker runs a container on **one machine**. Real systems need answers to questions Docker alone does not answer:

| Question | Kubernetes answer |
|---|---|
| A container crashes. Who restarts it? | Self-healing |
| Traffic grows. Who adds copies? | Scaling (manual or automatic) |
| A server dies. Where do its containers go? | Rescheduling on other nodes |
| How do I release a new version without downtime? | Rolling updates and rollbacks |
| How do containers find each other? | Services and DNS |
| Where do config and passwords live? | ConfigMaps and Secrets |
| Where does data live when a container dies? | Volumes, PV, PVC |

## Analogy: the orchestra conductor

Containers are musicians. Docker can start one musician. **Kubernetes is the conductor**: it decides who plays where, replaces a musician who walks off, and adds more players when the music needs them. You hand the conductor a score (your YAML), and it makes the orchestra match it.

## The core ideas (learn these 5 and the rest is detail)

1. **Declarative.** You describe *what you want* ("3 copies of my app"), not *how to do it*.
2. **Desired state vs actual state.** Your YAML is the desired state. The cluster reports the actual state.
3. **Reconciliation loop.** Controllers constantly compare the two and fix any difference.
4. **Everything is an API object.** Pods, Services, Deployments are all objects stored in the API server and managed with the same commands.
5. **Labels connect things.** Objects find each other by labels, not by names.

```
      you                         cluster
  ┌─────────┐   kubectl apply   ┌──────────────┐
  │  YAML   │ ────────────────► │  API server  │ ──► stored as DESIRED state
  └─────────┘                   └──────┬───────┘
                                       │
                          ┌────────────▼────────────┐
                          │  Controllers (loops)    │
                          │  desired  ≟  actual     │
                          │  different? → fix it    │
                          └─────────────────────────┘
```

## What Kubernetes is NOT

- Not a way to **build** images (use Docker, BuildKit, CI).
- Not a CI/CD system (it only runs what you deploy; GitLab CI or Argo CD deploy to it).
- Not a monitoring or logging stack (you add Prometheus, Loki, and so on).
- Not magic: it will not fix a broken application. It only restarts and reschedules it.

## First look (run these on your cluster)

```bash
kubectl cluster-info                  # where is the API server?
kubectl get nodes                     # the machines in the cluster
kubectl get pods -A                   # every Pod in every namespace
kubectl get pods -n kube-system       # Kubernetes' own components, running as Pods
```

## Gotchas

- Kubernetes manages **containers**, so you still need to understand Docker images first.
- It is a **platform for building platforms**, so expect to add other tools around it.

## Check yourself

1. What does "declarative" mean?
2. What is the reconciliation loop?
3. Name three problems Kubernetes solves that Docker alone does not.
4. Is Kubernetes a CI/CD tool?

<details>
<summary>Answers</summary>

1. You describe the end result you want, and the system figures out the steps
2. Controllers continuously compare actual state with desired state and correct any difference
3. Self-healing, scaling across machines, rolling updates, service discovery (any three)
4. No. It runs workloads. CI/CD tools deploy to it

</details>