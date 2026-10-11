# Startup Probe

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-autoscaling/manifests/`](../labs/07-autoscaling/manifests/) · Lab: [`labs/07-autoscaling`](../labs/07-autoscaling/)

[← Liveness Probe](liveness-probe.md) | [README](README.md) | [Horizontal Pod Autoscaler →](horizontal-pod-autoscaler.md)

---

## Idea

A **startup probe** answers: **"Has the application finished starting?"** While it has not succeeded yet, Kubernetes **holds off the liveness and readiness probes**. It exists to protect **slow-starting apps** from being killed by an impatient liveness probe.

```
Container starts
   │
   ▼
Startup probe runs ──(fails)──► keeps trying until failureThreshold is used up
   │                                      │
   │ succeeds                             └─► container is KILLED and restarted
   ▼
Startup probe never runs again
Liveness + Readiness probes take over (in parallel)
```

## The problem it solves

Without a startup probe, a slow app (large Java or Node app, one that runs migrations, loads a big cache) has two bad options:

| Option | Result |
|---|---|
| Short liveness settings | The app is **killed during startup**, restarted, killed again: `CrashLoopBackOff` |
| Huge `initialDelaySeconds` on liveness | The app starts fine, but a later **real hang takes minutes to detect**, because the delay applies only to the first check |

The startup probe gives you **both**: a long allowance to start, then strict liveness afterwards.

## YAML explained

```yaml
containers:
  - name: api
    image: my-api:1.0
    ports:
      - containerPort: 3000
    startupProbe:
      httpGet:
        path: /healthz
        port: 3000
      periodSeconds: 5
      failureThreshold: 30          # 30 × 5s = up to 150 seconds to start
    livenessProbe:
      httpGet:
        path: /healthz
        port: 3000
      periodSeconds: 10
      failureThreshold: 3           # strict: 30 seconds to detect a hang
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      periodSeconds: 5
```

**Maximum start time = `failureThreshold × periodSeconds`** (plus `initialDelaySeconds` if set). If the app has not passed by then, the container is **killed** and restarted.

| Field | Meaning |
|---|---|
| Handlers | `httpGet`, `tcpSocket`, `exec`, `grpc` (same as the others) |
| `periodSeconds` | How often to check during startup |
| `failureThreshold` | How many failures to tolerate: this sets your total allowance |
| `successThreshold` | Must be 1 |
| `terminationGracePeriodSeconds` | Optional override for shutdown after a startup failure |

### Choosing the numbers

```
Measure how long your app takes to be ready (p99 across environments), then add margin.

Typical start 40s, worst case 90s  →  periodSeconds: 5, failureThreshold: 30  (150s allowance)
```

A larger allowance does not slow healthy starts: the probe succeeds on the **first** passing check and then stops.

## The three probes together

| Probe | Question | Runs when | Failure |
|---|---|---|---|
| **Startup** | "Have you started?" | Only at startup, until it passes once | Container **restarted** |
| **Liveness** | "Are you stuck?" | After startup succeeded | Container **restarted** |
| **Readiness** | "Can you serve now?" | After startup succeeded, always | Removed from **endpoints** |

Order: **startup first**, then **liveness and readiness together**. Until startup succeeds, the Pod is not Ready.

## When you do not need one

- The app is ready within a few seconds and a normal `initialDelaySeconds` is enough.
- The app has no liveness probe and only needs readiness.

If you find yourself setting `initialDelaySeconds` above about 30 seconds on a liveness probe, use a startup probe instead.

## Lab

```bash
cd labs/07-autoscaling/manifests
kubectl apply -f startup-demo.yaml
```

Two Pods run the same "slow app" that needs 45 seconds to start. Both have a strict liveness probe (one failure kills it). Only one has a startup probe.

**1. Without a startup probe: killed during startup**

```bash
kubectl get pod startup-without -w                     # restarts again and again, CrashLoopBackOff
kubectl describe pod startup-without | grep -i "liveness probe failed"
```

**2. With a startup probe: survives, then becomes Ready**

```bash
kubectl get pod startup-with -w                        # Running 0/1 for ~45s, then 1/1, RESTARTS 0
kubectl describe pod startup-with | grep -i -E "startup|unhealthy"     # Startup probe failed while starting, then it stops
```

**3. Liveness is strict after startup**

```bash
kubectl exec startup-with -- rm /tmp/started           # simulate a hang after a successful start
kubectl get pod startup-with -w                        # liveness fails and the container is restarted quickly
```

**4. Make the allowance too small**

Edit `startup-with` in the YAML: `failureThreshold: 5` (25 seconds, less than 45). Delete and recreate it. The container is killed before it finishes starting. This is what happens if you under-estimate startup time.

**Cleanup**

```bash
kubectl delete -f startup-demo.yaml
```

## Gotchas

- A **too small allowance** (`failureThreshold × periodSeconds`) restarts the app forever. Measure real startup time.
- The startup probe runs **only until its first success**. It does not monitor later health.
- Until it succeeds, **readiness and liveness are disabled**, so the Pod is simply not Ready.
- The probe is per **container**. With several containers, each can have its own.
- Startup probes need Kubernetes 1.20 or newer (they have been stable for a long time).
- A startup probe does not make a slow app faster. Also consider lazy loading and smaller images.

## Check yourself

1. What problem does a startup probe solve?
2. Which probes are disabled until the startup probe succeeds?
3. How do you calculate the maximum startup time?
4. Does a larger `failureThreshold` slow down apps that start quickly?
5. What happens if the startup probe never succeeds?

<details>
<summary>Answers</summary>

1. It gives slow-starting apps a long allowance to start without having to weaken the liveness probe
2. Liveness and readiness
3. `failureThreshold × periodSeconds` (plus `initialDelaySeconds`)
4. No. The probe succeeds on the first passing check and then stops
5. After the allowance is used up the container is killed and restarted

</details>

---

[← Liveness Probe](liveness-probe.md) | [README](README.md) | [Horizontal Pod Autoscaler →](horizontal-pod-autoscaler.md)