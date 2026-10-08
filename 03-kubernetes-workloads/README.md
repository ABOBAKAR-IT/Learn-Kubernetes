# Module 03: Kubernetes Workloads

> Run applications: Pods, Deployments, ReplicaSets, DaemonSets, StatefulSets, Jobs and CronJobs.

**Notes ready:** 7/7 topics

## Topic directory

| Topic | File | Question it answers | Status |
|---|---|---|---|
| **Pods** | [`pods.md`](pods.md) | What is the smallest unit, how does it live and die? | 📘 |
| **Deployments** | [`deployments.md`](deployments.md) | How do I update and roll back without downtime? | 📘 |
| **ReplicaSets** | [`replicasets.md`](replicasets.md) | How are N copies kept alive? | 📘 |
| **DaemonSets** | [`daemonsets.md`](daemonsets.md) | How do I run one Pod per node? | 📘 |
| **StatefulSets** | [`statefulsets.md`](statefulsets.md) | How do Pods get stable names and their own storage? | 📘 |
| **Jobs** | [`jobs.md`](jobs.md) | How do I run something once, to completion? | 📘 |
| **CronJobs** | [`cronjobs.md`](cronjobs.md) | How do I run it on a schedule? | 📘 |

YAML files for every lab: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/).

## Recommended study order

| # | Topic | Why now |
|---|---|---|
| 1 | [Pods](pods.md) | Everything else wraps Pods |
| 2 | [ReplicaSets](replicasets.md) | The engine that keeps N Pods alive |
| 3 | [Deployments](deployments.md) | The controller you will use most |
| 4 | [DaemonSets](daemonsets.md) | One Pod per node |
| 5 | [StatefulSets](statefulsets.md) | Stable identity and storage |
| 6 | [Jobs](jobs.md) | Run to completion |
| 7 | [CronJobs](cronjobs.md) | Jobs on a schedule |

ReplicaSets come before Deployments because a Deployment creates ReplicaSets, so you understand what it manages.

Every workload controller does the same job: it **creates Pods from a template**. They differ only in policy.

```
                      ┌───────────────┐
Deployment   ────────►│               │ stateless apps, rolling updates
StatefulSet  ────────►│               │ stable names + own storage
DaemonSet    ────────►│  Pod template │ one per node
Job          ────────►│  ──► Pods     │ run to completion
CronJob      ────────►│               │ Job on a schedule
(ReplicaSet: the engine inside a Deployment)
                      └───────────────┘
```

## Choosing the right workload

Every workload controller does the same basic thing: it **creates Pods from a template**. They differ in **policy**: how many, where, how long, with what identity.

```
Deployment ─┐
StatefulSet ┤
DaemonSet   ┼──► Pod template ──► Pods
Job         ┤
CronJob ────┘ (via Job)
```

### Decision flow

```
Does the work run forever?
│
├─ NO ── Run once, then stop?            → Job
│        Run on a schedule?              → CronJob
│
└─ YES ─ One Pod on every node?          → DaemonSet
         Needs stable name, own storage,
         or ordered start/stop?          → StatefulSet
         Otherwise (stateless)           → Deployment

A bare Pod: only for quick tests and debugging.
```

### Comparison

| | Deployment | StatefulSet | DaemonSet | Job | CronJob |
|---|---|---|---|---|---|
| Runs | Forever | Forever | Forever | To completion | On a schedule |
| Pod names | Random | `name-0, name-1...` | Random | Random | Random |
| Replicas | `replicas` | `replicas` | One per node | `completions` / `parallelism` | Per run |
| Storage | Shared or none | **One PVC per Pod** | Usually hostPath | Optional | Optional |
| Update | Rolling, rollback | Rolling (reverse order), partition | Rolling | Not updatable | New runs use new spec |
| restartPolicy | Always | Always | Always | OnFailure / Never | OnFailure / Never |
| Creates | ReplicaSets | Pods | Pods | Pods | Jobs |

### Typical examples

| You have... | Use |
|---|---|
| REST API, web frontend, stateless worker | **Deployment** |
| Database, Kafka, Elasticsearch, anything needing stable identity and disk | **StatefulSet** (or a managed service) |
| Log collector, monitoring agent, network plugin | **DaemonSet** |
| Database migration, one-off import, batch conversion | **Job** |
| Nightly backup, hourly report, periodic cleanup | **CronJob** |

### A common real-world app

```
Deployment  api            (3 replicas, behind a Service)
Deployment  worker         (consumes a queue, scales with load)
StatefulSet postgres       (or use a managed database)
Job         migrate        (run before each release)
CronJob     nightly-report (02:00)
DaemonSet   log-agent      (one per node, installed by the platform team)
```

### Rules of thumb

1. **Default to Deployment.** Reach for others only when you need what they add.
2. **Never run bare Pods** for real work. Nothing restarts them.
3. **Avoid StatefulSets for databases in production** unless you also have an operator, backups and a failover plan.
4. **Keep Jobs and CronJobs idempotent** and give them a TTL.
5. **Do not mix concerns:** a migration is a Job, not an init container of the API (every replica would run it).

### Check yourself

Pick the workload:

1. A NestJS API that must stay up with 3 copies.
2. A Redis cluster where each member keeps its own data.
3. A Fluent Bit log shipper on every node.
4. A script that imports a CSV once.
5. An email digest sent at 08:00 every weekday.

<details>
<summary>Answers</summary>

1. Deployment
2. StatefulSet
3. DaemonSet
4. Job
5. CronJob (`0 8 * * 1-5`)

</details>

## Related labs and references

- [labs/01-first-pod](../labs/01-first-pod/): create, inspect, break and fix a Pod
- [labs/02-deployment](../labs/02-deployment/): capstone with Deployment, rollback and DaemonSet, plus all manifests
- [cheatsheets/building-blocks.md](../cheatsheets/building-blocks.md): quick map of Node, Namespace, Pod, Labels, controllers, Service

```bash
# Stateless app
kubectl create deployment api --image=nginx:1.27 --replicas=3

# Run-to-completion work
kubectl create job migrate --image=busybox:1.36 -- sh -c "echo migrating; sleep 3"

# Scheduled work
kubectl create cronjob report --image=busybox:1.36 --schedule="*/1 * * * *" -- sh -c "date"

# Stateful app: see statefulsets.md

kubectl get deploy,sts,ds,job,cronjob,pods
kubectl delete deployment api; kubectl delete job migrate; kubectl delete cronjob report
```

## Module quiz

### Part 1: Pods, labels, ReplicaSets, Deployments, DaemonSets

1. What does a Pod's containers share?
2. Why do we not run bare Pods in production?
3. What are the two selector types, and which controllers support them?
4. What does a ReplicaSet do when a Pod dies?
5. What does a Deployment create when you apply it?
6. What triggers a new Revision: scaling, labeling the Deployment, or changing the image?
7. After a rolling update, what state is the old ReplicaSet in, and why?
8. How do you roll back to Revision 1?
9. What is the default Deployment update strategy, and what is the other one?
10. What is special about how a DaemonSet places Pods?
11. Name two DaemonSet use cases.
12. Why does a DaemonSet manifest have no `replicas`?
13. In a nested Pod template, which two fields are dropped?
14. What command generates a Pod YAML without creating the Pod?

<details>
<summary>Answers</summary>

1. The same network namespace (one IP), the same node, and access to the same volumes
2. They are ephemeral and have no self-healing; controllers provide replication and recovery
3. Equality-based (`=`, `==`, `!=`) and set-based (`in`, `notin`, exists). RC: equality only. ReplicaSet, Deployment, DaemonSet: both. Services use selectors too
4. Notices current ≠ desired and creates a replacement Pod
5. A ReplicaSet, which creates the Pods
6. Only changing the image (a Pod template change). Scaling and labeling do not
7. Scaled to 0 Pods but kept, so it can be used for rollback (Revisions)
8. `kubectl rollout undo deploy <name> --to-revision=1`
9. RollingUpdate (default); Recreate is the disruptive alternative
10. Exactly one Pod per node (all nodes or a subset), instead of "N Pods anywhere"
11. Log/monitoring agents, storage, networking or proxy daemons (for example kube-proxy, Calico/Cilium agents)
12. The number of matching nodes decides the Pod count
13. `apiVersion` and `kind`
14. `kubectl run <name> --image=<img> --dry-run=client -o yaml`

</details>

---

### Part 2: Pod lifecycle, StatefulSets, Jobs, CronJobs

1. Is `CrashLoopBackOff` a Pod phase?
2. What does exit code 137 usually mean?
3. What is the order in which init containers run?
4. Why can `CMD npm start` slow down Pod shutdown?
5. What is the Pod name pattern of a StatefulSet?
6. Why does a StatefulSet need a headless Service?
7. What happens to PVCs when a StatefulSet is scaled down or deleted?
8. Which `restartPolicy` values can a Job use?
9. What do `completions` and `parallelism` control?
10. What does `concurrencyPolicy: Forbid` do?
11. How do you run a CronJob once, right now?
12. Which workload fits: a database migration before each release? A log shipper on every node? A stateless API?

<details>
<summary>Answers</summary>

1. No. It is a container state reason; the phase stays `Running` or `Pending`
2. SIGKILL: out of memory (`OOMKilled`) or the grace period expired
3. One at a time, in order; each must succeed before the next, and all finish before the app containers start
4. npm does not forward SIGTERM to your app, so Kubernetes waits the whole grace period, then sends SIGKILL
5. `<name>-0`, `<name>-1`, ... (stable ordinals)
6. It gives each Pod its own stable DNS name such as `web-0.web`
7. The PVCs are kept (by default) and must be deleted manually
8. `Never` and `OnFailure`
9. `completions`: how many Pods must succeed. `parallelism`: how many run at once
10. Skips the new run if the previous run is still going
11. `kubectl create job <name> --from=cronjob/<cronjob-name>`
12. Job; DaemonSet; Deployment

</details>

---

### Part 3: Extra practice on StatefulSets, Jobs and CronJobs

1. What are three things a StatefulSet guarantees that a Deployment does not?
2. Which kind of Service does a StatefulSet need, and why?
3. What are the PVC names for a StatefulSet `db` with a `volumeClaimTemplate` named `data` and 3 replicas?
4. You scale a StatefulSet from 3 to 1. Which Pods stop, in which order, and what happens to their PVCs?
5. Does a StatefulSet replicate data between Pods?
6. Which `restartPolicy` values are valid for a Job?
7. How do you run 12 tasks, 4 at a time?
8. A Job's Pods all fail. What stops the retries?
9. What does `concurrencyPolicy: Replace` do?
10. How do you trigger a CronJob right now?
11. Which workload would you use for: (a) a REST API, (b) a log collector on every node, (c) PostgreSQL with its own disk, (d) a nightly backup, (e) a one-time data migration?

<details>
<summary>Answers</summary>

1. Stable names, per-Pod storage, ordered start/stop/update (also stable DNS per Pod)
2. A headless Service (`clusterIP: None`), so each Pod gets its own DNS name
3. `data-db-0`, `data-db-1`, `data-db-2`
4. `db-2` then `db-1` stop. Their PVCs are kept
5. No. The application or an operator does that
6. `Never` and `OnFailure`
7. `completions: 12`, `parallelism: 4`
8. `backoffLimit` (default 6). After that the Job is marked Failed
9. If the previous run is still running, it is stopped and replaced by the new run
10. `kubectl create job <name> --from=cronjob/<cronjob-name>`
11. (a) Deployment, (b) DaemonSet, (c) StatefulSet, (d) CronJob, (e) Job

</details>

## Self-test before moving on

1. For each of five real tasks, name the workload and say why.
2. Explain why a StatefulSet keeps its PVCs after being deleted.
3. Show what happens to a Pod when you delete it under a Deployment vs a bare Pod.
4. Take the module quiz above without looking at the answers.

## How to study this module

1. Read each note in order.
2. Type the YAML yourself, do not copy-paste.
3. Run the lab, then break it on purpose.
4. When you have studied and practiced a note, change its `Status` line to `✅ Studied`.

Next: [Module 04: Configuration](../04-kubernetes-configuration/) or, in the recommended order, [Module 05: Networking](../05-kubernetes-networking/)