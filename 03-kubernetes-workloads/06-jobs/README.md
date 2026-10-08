# Jobs

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/)

[← StatefulSets](statefulsets.md) | [README](README.md) | [CronJobs →](cronjobs.md)

---

## Idea

A **Job** runs Pods **to completion**. Deployments keep Pods running forever, but a Job runs a task, waits for it to **succeed**, and then stops.

Use it for: database migrations, batch processing, report generation, backups, one-off scripts, training runs.

```
Job ─► Pod runs the task ─► exit code 0 ─► Job "Complete"
              │
              └─ exit code ≠ 0 ─► retry (up to backoffLimit) ─► Job "Failed"
```

## Rules for the Pod template

| Rule | Why |
|---|---|
| `restartPolicy` must be **`Never`** or **`OnFailure`** | `Always` would restart a finished task forever |
| `Never`: a failed Pod stays (for logs) and a **new Pod** is created for the retry | Easy to inspect failures |
| `OnFailure`: the **same Pod** restarts the failing container | Fewer Pods, less history |

## Key fields

| Field | Default | Meaning |
|---|---|---|
| `completions` | 1 | How many Pods must finish successfully |
| `parallelism` | 1 | How many Pods may run at the same time |
| `backoffLimit` | 6 | Retries before the Job is marked Failed (delay grows: 10s, 20s, 40s...) |
| `activeDeadlineSeconds` | none | Maximum run time for the whole Job |
| `ttlSecondsAfterFinished` | none | Auto-delete the Job and its Pods this long after it ends |
| `completionMode` | NonIndexed | `Indexed` gives each Pod a number in `JOB_COMPLETION_INDEX` |
| `suspend` | false | Pause the Job without deleting it |

## Three Job patterns

| Pattern | Settings | Example |
|---|---|---|
| **Single task** | defaults (1 completion) | Run one migration |
| **Fixed number of tasks** | `completions: 6`, `parallelism: 2` | Process 6 files, 2 at a time |
| **Indexed work split** | `completionMode: Indexed` | Pod N handles slice N of the data |

## YAML examples

**Single task**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: hello
          image: busybox:1.36
          command: ["sh", "-c", "echo Hello from a Job; date"]
```

**Parallel: 6 tasks, 2 at a time**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 6
  parallelism: 2
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo processing on $(hostname); sleep 10"]
```

**A Job that always fails (to see retries)**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: fail-job
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: fail
          image: busybox:1.36
          command: ["sh", "-c", "echo trying...; exit 1"]
```

**Indexed**

```yaml
spec:
  completions: 3
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo I am slice $JOB_COMPLETION_INDEX"]
```

## Commands

```bash
kubectl create job hello --image=busybox:1.36 -- echo hello    # imperative
kubectl get jobs
kubectl get pods -l job-name=hello-job
kubectl logs job/hello-job
kubectl describe job hello-job
kubectl wait --for=condition=complete job/hello-job --timeout=60s
kubectl delete job hello-job                                   # also deletes its Pods
```

## Lab

```bash
# 1. Single task
kubectl apply -f hello-job.yaml
kubectl get jobs                                   # COMPLETIONS 0/1 then 1/1
kubectl get pods                                   # STATUS Completed
kubectl logs job/hello-job

# 2. Parallel: watch two at a time
kubectl apply -f parallel-job.yaml
kubectl get pods -l job-name=parallel-job -w       # 2 running, then the next 2, then the last 2
kubectl get job parallel-job                       # COMPLETIONS 6/6

# 3. Failure and retries
kubectl apply -f fail-job.yaml
kubectl get pods -l job-name=fail-job -w           # several Error Pods, with growing delays
kubectl describe job fail-job | tail -8            # BackoffLimitExceeded

# 4. Indexed
kubectl logs -l job-name=indexed-job --prefix      # each Pod prints a different slice number

# Cleanup
kubectl delete job hello-job parallel-job fail-job indexed-job --ignore-not-found
```

## Gotchas

- **A task may run more than once.** Retries and node failures mean "at least once", so make tasks **idempotent** (running twice causes no harm).
- Finished Jobs and their Pods **stay forever** unless you set `ttlSecondsAfterFinished`. They clutter `kubectl get pods`.
- A Job's Pod template is **immutable**. To change it, delete and recreate the Job.
- `restartPolicy: Always` is rejected for Jobs.
- `activeDeadlineSeconds` applies to the **whole Job**, not each Pod.

## Check yourself

1. How does a Job differ from a Deployment?
2. Which `restartPolicy` values are allowed in a Job?
3. How do you run 10 tasks, 3 at a time?
4. What happens when `backoffLimit` is exceeded?
5. Why should Job tasks be idempotent?

<details>
<summary>Answers</summary>

1. A Job runs until the task succeeds then stops. A Deployment keeps Pods running indefinitely
2. `Never` and `OnFailure`
3. `completions: 10`, `parallelism: 3`
4. The Job is marked Failed (`BackoffLimitExceeded`) and stops creating Pods
5. Retries or failures can run the same task more than once

</details>

---

[← StatefulSets](statefulsets.md) | [README](README.md) | [CronJobs →](cronjobs.md)