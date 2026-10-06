# Why Kubernetes?

> Status: 📘 Notes ready

## How we got here

```
Physical servers  →  Virtual machines  →  Containers  →  Container orchestration
 1 app per box        many VMs per box     many apps       many containers across
 slow, wasteful       heavy, slow boot     lightweight     many machines, managed
```

| Era | Strength | Weakness |
|---|---|---|
| Physical servers | Simple | One app per machine, waste, slow to provision |
| Virtual machines | Isolation, better use of hardware | Each VM carries a full OS, slow to start |
| Containers | Small, fast, same everywhere | Someone must place, restart and connect them |
| **Orchestration** | Automates all of that | More moving parts to learn |

Containers made it easy to package an app. Orchestration makes it easy to **run hundreds of them reliably**.

## What you gain

| Benefit | What it means in practice |
|---|---|
| Self-healing | Crashed containers restart, dead nodes are replaced |
| Horizontal scaling | `replicas: 3` becomes `replicas: 30` |
| Zero-downtime releases | Rolling update, roll back with one command |
| Portability | The same YAML runs on a laptop, on AWS, on-premises |
| Efficient hardware use | Many apps packed onto shared machines |
| Standard API | One set of tools (`kubectl`, Helm, GitOps) everywhere |
| Huge ecosystem | Monitoring, security, networking and CI/CD tools built for it |

## What it costs

- **Complexity:** many concepts (Pods, Services, Ingress, RBAC...).
- **Learning curve:** YAML, networking, and debugging across layers.
- **Operational work:** upgrades, security, monitoring (less with managed services such as EKS).
- **Overkill for small apps:** a single small service rarely needs it.

## When to use it and when not to

| Use Kubernetes when... | Skip it when... |
|---|---|
| Many services or teams | One or two small apps |
| You need scaling and high availability | Traffic is tiny and steady |
| You deploy often | You deploy rarely |
| You want portability across clouds | You are happy locked into one platform |
| You run long-lived or stateful services | Short event-driven functions fit (for example AWS Lambda) |

## Alternatives at a glance

| Tool | Best for | Trade-off |
|---|---|---|
| Docker Compose | Local dev, one machine | No multi-node, no self-healing across machines |
| Docker Swarm | Simple clustering | Small ecosystem, much less used today |
| AWS ECS (and Fargate) | Containers on AWS with less to learn | AWS-only |
| HashiCorp Nomad | Mixed workloads, simpler | Smaller ecosystem |
| Serverless (Lambda) | Event-driven code, no servers | Limits on runtime, state and control |
| **Kubernetes** | Portable, flexible, large ecosystem | Most complex |

A common real-world split: **Lambda for event-driven pieces, Kubernetes for long-running services**.

## Gotchas

- "Everyone uses it" is not a reason. Match the tool to the problem.
- Managed Kubernetes (EKS, GKE, AKS) removes control plane work, but you still run the workloads.

## Check yourself

1. Why were containers not enough on their own?
2. Name three benefits and two costs of Kubernetes.
3. When would Docker Compose or Lambda be a better choice?

<details>
<summary>Answers</summary>

1. Nobody places them on machines, restarts them, scales them or connects them. Orchestration does
2. Benefits: self-healing, scaling, zero-downtime updates, portability. Costs: complexity, learning curve, operational work
3. Compose: local development or a single small host. Lambda: short event-driven functions with no servers to manage

</details>