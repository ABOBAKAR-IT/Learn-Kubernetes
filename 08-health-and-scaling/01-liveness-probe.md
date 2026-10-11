# Liveness Probe

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-autoscaling/manifests/`](../labs/07-autoscaling/manifests/) · Lab: [`labs/07-autoscaling`](../labs/07-autoscaling/)

[← Readiness Probe](readiness-probe.md) | [README](README.md) | [Startup Probe →](startup-probe.md)

---

## Idea

A **liveness probe** answers: **"Is this container stuck beyond recovery?"** If the probe keeps failing, the kubelet **kills the container and restarts it**.

```
Liveness passes ─► nothing happens
Liveness fails (failureThreshold times in a row) ─► container is killed ─► restarted per restartPolicy
```

It is for failures a **restart can fix**: deadlocks, infinite loops, a hung event loop, an app that is running but no longer responds. A crashed process is already restarted by Kubernetes without a probe.

## YAML explained

```yaml
containers:
  - name: api
    image: my-api:1.0
    livenessProbe:
      httpGet:
        path: /healthz
        port: 3000
      initialDelaySeconds: 10    # grace period before the first check
      periodSeconds: 10          # check every 10 seconds
      timeoutSeconds: 2          # a slow answer counts as a failure
      failureThreshold: 3        # 3 failures in a row → restart
```

Time before a restart: about **`failureThreshold × periodSeconds`** after the app actually hangs (here 30 seconds).

### The four handlers

```yaml
# HTTP: success = status 200-399
livenessProbe:
  httpGet: { path: /healthz, port: 3000 }

# TCP: success = the port accepts a connection
livenessProbe:
  tcpSocket: { port: 5432 }

# Exec: success = exit code 0
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]

# gRPC: success = the gRPC health check returns SERVING
livenessProbe:
  grpc:
    port: 9000
```

### Fields

| Field | Default | Meaning |
|---|---|---|
| `initialDelaySeconds` | 0 | Wait before the first probe |
| `periodSeconds` | 10 | Probe interval |
| `timeoutSeconds` | 1 | Probe timeout |
| `successThreshold` | 1 | Must be **1** for liveness |
| `failureThreshold` | 3 | Failures in a row before a restart |
| `terminationGracePeriodSeconds` | Pod's value | Override the shutdown grace time for a restart caused by this probe |

## What a good liveness check looks like

| Do | Do not |
|---|---|
| Check that **this process** can respond | Check **databases or other services** |
| Keep it **cheap** and fast | Run heavy queries |
| Use a **separate endpoint** from readiness | Reuse the readiness check |
| Use `failureThreshold` of 3 or more | Restart on a single blip |
| Use a [startup probe](startup-probe.md) for slow starts | Guess a huge `initialDelaySeconds` |

### Why not check dependencies?

If liveness fails when the database is down, **every Pod is killed and restarted** while the database is down. Restarts do not fix a database. They add load and turn a partial outage into a complete one. Dependency checks belong in [readiness](readiness-probe.md), where the effect is "stop sending traffic", not "kill the app".

### Minimal examples

```js
// Express
app.get('/healthz', (req, res) => res.status(200).send('ok'));
```

```ts
// NestJS: no dependencies in the liveness check
@Get('live') live() { return this.health.check([]); }
```

If this endpoint answers, the event loop is alive. If the event loop is blocked, it does not answer, and Kubernetes restarts the Pod.

## What happens on a failure

1. The probe fails `failureThreshold` times in a row.
2. The kubelet logs an event: `Liveness probe failed: ...`.
3. It sends SIGTERM, waits the grace period, then SIGKILL.
4. The container restarts (`restartPolicy: Always`), and `RESTARTS` increases.
5. Repeated restarts are delayed with a back-off: `CrashLoopBackOff`.

## Lab

```bash
cd labs/07-autoscaling/manifests
```

**1. Exec probe: a file the app removes**

```bash
kubectl apply -f liveness-exec.yaml
kubectl get pod liveness-exec -w                      # healthy for ~30s, then restarts
kubectl describe pod liveness-exec | grep -A8 Events  # "Liveness probe failed: cat: can't open '/tmp/healthy'"
kubectl get pod liveness-exec                          # RESTARTS 1, 2, ...
```

**2. HTTP probe: break a running web server**

```bash
kubectl apply -f liveness-http.yaml
kubectl get pod liveness-http                          # Running, RESTARTS 0
kubectl exec liveness-http -- rm /usr/share/nginx/html/index.html      # now / returns 403
kubectl get pod liveness-http -w                       # within ~15s the container is restarted
kubectl describe pod liveness-http | grep -i "liveness probe failed"   # HTTP probe failed with statuscode: 403
kubectl exec liveness-http -- ls /usr/share/nginx/html                 # a fresh container: index.html is back
```

**3. See the thresholds work**

Edit `liveness-http.yaml`, set `failureThreshold: 6`, recreate the Pod and repeat step 2. The restart now takes noticeably longer.

**Cleanup**

```bash
kubectl delete -f liveness-exec.yaml -f liveness-http.yaml
```

## Gotchas

- **Do not check dependencies** in liveness. A failing database should not make you restart healthy apps.
- **Too aggressive probes** (low threshold, 1 second timeout) restart apps under load, which makes the load worse.
- A liveness probe that is too strict for a **slow start** causes a restart loop. Add a startup probe.
- The probe needs `timeoutSeconds` above your worst healthy response time, including garbage-collection pauses.
- Restarting does not fix **bad data or config**. If the app is broken at startup, you get `CrashLoopBackOff`.
- A liveness probe is optional. Use it only when a restart can actually help.

## Check yourself

1. What does the kubelet do when a liveness probe fails `failureThreshold` times?
2. Why should a liveness check avoid calling the database?
3. How long after a hang is the container restarted, if `periodSeconds: 10` and `failureThreshold: 3`?
4. Which `successThreshold` is allowed for liveness?
5. What should you use instead of a large `initialDelaySeconds` for slow-starting apps?

<details>
<summary>Answers</summary>

1. It kills the container and restarts it according to the Pod's `restartPolicy`
2. If the database is down, all Pods would be restarted for nothing and a partial outage becomes a total one
3. About 30 seconds
4. Only 1
5. A startup probe

</details>

---

[← Readiness Probe](readiness-probe.md) | [README](README.md) | [Startup Probe →](startup-probe.md)