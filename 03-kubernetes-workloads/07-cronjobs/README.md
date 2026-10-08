# CronJobs

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/02-deployment/manifests/`](../labs/02-deployment/manifests/)

[← Jobs](jobs.md) | [README](README.md) | [Module README →](README.md)

---

## Idea

A **CronJob** creates **Jobs on a schedule**, like Linux `cron`. Each run is a normal Job with its own Pod(s).

```
CronJob ──(on schedule)──► Job ──► Pod runs the task
   │
   └─ keeps a short history of finished Jobs
```

Use it for: backups, nightly reports, cleanup tasks, cache warming, periodic syncs.

## Schedule syntax

```
┌───────── minute        (0-59)
│ ┌─────── hour          (0-23)
│ │ ┌───── day of month  (1-31)
│ │ │ ┌─── month         (1-12)
│ │ │ │ ┌─ day of week   (0-6, Sunday = 0)
* * * * *
```

| Schedule | Meaning |
|---|---|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` (or `@hourly`) | Every hour, on the hour |
| `0 2 * * *` (or daily at 02:00) | Every day at 02:00 |
| `0 9 * * 1-5` | Weekdays at 09:00 |
| `30 3 * * 0` | Sundays at 03:30 |
| `0 0 1 * *` (or `@monthly`) | First day of each month at midnight |

## Key fields

| Field | Default | Meaning |
|---|---|---|
| `schedule` | required | Cron expression |
| `timeZone` | controller manager's zone (often UTC) | For example `Etc/UTC` or `Asia/Karachi`. Needs Kubernetes 1.27 or newer |
| `concurrencyPolicy` | `Allow` | What if the previous run is still going: `Allow`, `Forbid` (skip the new run), `Replace` (stop the old run, start the new one) |
| `startingDeadlineSeconds` | none | How late a run may start before it is counted as missed |
| `successfulJobsHistoryLimit` | 3 | Finished Jobs to keep |
| `failedJobsHistoryLimit` | 1 | Failed Jobs to keep |
| `suspend` | false | Pause the schedule |
| `jobTemplate` | required | A Job spec (everything from the Jobs note applies) |

## YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"              # every minute (for the lab)
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 60
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 300
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: hello
              image: busybox:1.36
              command: ["sh", "-c", "date; echo Hello from the CronJob"]
```

The nesting is: **CronJob `spec` → `jobTemplate.spec` (Job) → `template.spec` (Pod)**.

## Commands

```bash
kubectl create cronjob hello-cron --image=busybox:1.36 --schedule="*/1 * * * *" -- echo hi
kubectl get cronjob                                   # LAST SCHEDULE, ACTIVE
kubectl get jobs
kubectl create job manual-run --from=cronjob/hello-cron      # run it right now
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":true}}'    # pause
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":false}}'   # resume
kubectl delete cronjob hello-cron                     # also deletes its Jobs
```

## Lab

```bash
# 1. Create and watch
kubectl apply -f hello-cron.yaml
kubectl get cronjob hello-cron
kubectl get jobs -w                                   # a new Job appears every minute
kubectl logs job/<job-name>                           # the output of one run

# 2. Run it on demand (great for testing a scheduled task)
kubectl create job manual-run --from=cronjob/hello-cron
kubectl logs job/manual-run

# 3. Pause and resume
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":true}}'
kubectl get cronjob                                   # SUSPEND True, no new Jobs
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":false}}'

# 4. History limit in action: after 5+ minutes
kubectl get jobs                                      # only the last 3 successful runs remain

# 5. Overlap: make the task longer than the schedule
kubectl patch cronjob hello-cron --type=json -p='[{"op":"replace","path":"/spec/jobTemplate/spec/template/spec/containers/0/command","value":["sh","-c","echo start; sleep 90; echo done"]}]'
kubectl get jobs -w                                   # with Forbid, the next run is skipped while one is active

# Cleanup
kubectl delete cronjob hello-cron
kubectl delete job manual-run --ignore-not-found
```

## Gotchas

- **Not exactly once.** A run can be skipped (cluster down, `Forbid`) or rarely start twice. Make tasks idempotent.
- **Time zones.** Without `timeZone` the schedule uses the controller manager's zone, usually UTC. Daylight saving time can skip or repeat runs.
- **Name length.** A CronJob name is limited to 52 characters because the Job name adds a suffix.
- **Missed runs.** If more than 100 runs are missed (and `startingDeadlineSeconds` is unset), the controller stops scheduling and logs an error.
- **Clutter.** Keep `ttlSecondsAfterFinished` and the history limits small.
- Cluster-level scheduling is similar in spirit to a cloud scheduler (such as an EventBridge schedule triggering a task), except here the task runs as a Pod.

## Check yourself

1. What object does a CronJob create on each run?
2. What does `0 2 * * *` mean?
3. What does `concurrencyPolicy: Forbid` do?
4. How do you run a CronJob's task immediately for testing?
5. Why should the scheduled task be idempotent?

<details>
<summary>Answers</summary>

1. A Job (which then creates Pods)
2. Every day at 02:00
3. If the previous run is still active, the new run is skipped
4. `kubectl create job <name> --from=cronjob/<cronjob-name>`
5. A run can be missed or, rarely, run twice, so repeating it must be safe

</details>

---

[← Jobs](jobs.md) | [README](README.md) | [Module README →](README.md)