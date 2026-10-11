# Readiness Probe

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-autoscaling/manifests/`](../labs/07-autoscaling/manifests/) · Lab: [`labs/07-autoscaling`](../labs/07-autoscaling/)

[README](README.md) | [Liveness Probe →](liveness-probe.md)

---

## Idea

A **readiness probe** answers: **"Can this container receive traffic right now?"**

- **Passing**: the Pod is `Ready` and is included in its Services' endpoints.
- **Failing**: the Pod becomes **not Ready** and is **removed from the endpoints**. It is **not restarted**.

```
Readiness fails  ─►  Pod marked NotReady  ─►  removed from Service endpoints  ─►  no new traffic
Readiness passes ─►  Pod marked Ready     ─►  added back
```

It runs for the **whole life** of the container, so a Pod can go in and out of rotation many times without restarts.

## What depends on readiness

| Consumer | Effect |
|---|---|
| **Services** (EndpointSlices) | Only Ready Pods receive traffic |
| **Ingress / load balancers** | Targets are Ready Pods (AWS ALB/NLB target health can add its own readiness gate) |
| **Rolling updates** | A Deployment waits for new Pods to be Ready before removing old ones |
| **`kubectl rollout status`** | Reports success only when Pods are Ready |
| **PodDisruptionBudgets** | Count only Ready Pods as available |
| **HPA** | Ignores metrics from Pods that are not yet Ready (they are still warming up) |

Without a readiness probe, a Pod is considered Ready **as soon as the container starts**, which is usually too early.

## YAML explained

```yaml
containers:
  - name: api
    image: my-api:1.0
    ports:
      - containerPort: 3000
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5     # wait before the first check
      periodSeconds: 5           # check every 5 seconds
      timeoutSeconds: 2          # each check must answer within 2 seconds
      successThreshold: 1        # successes needed to become Ready again
      failureThreshold: 3        # consecutive failures before NotReady
```

### Probe handlers

| Handler | Success means | Example |
|---|---|---|
| `httpGet` | HTTP status **200 to 399** | `path: /ready`, `port: 3000` |
| `tcpSocket` | The TCP connection opens | `port: 5432` |
| `exec` | The command exits with **0** | `command: ["cat", "/tmp/ready"]` |
| `grpc` | The gRPC health check returns `SERVING` | `port: 9000` (and optional `service`) |

### Timing fields

| Field | Default | Meaning |
|---|---|---|
| `initialDelaySeconds` | 0 | Delay before the first probe |
| `periodSeconds` | 10 | How often to probe |
| `timeoutSeconds` | 1 | Per-probe timeout |
| `successThreshold` | 1 | Consecutive successes to count as success (can be above 1 for readiness only) |
| `failureThreshold` | 3 | Consecutive failures to count as failure |

Time to be marked NotReady after a real failure: about `failureThreshold × periodSeconds`.

## Readiness vs liveness

| | Readiness | [Liveness](liveness-probe.md) |
|---|---|---|
| Question | "Can you serve traffic **now**?" | "Are you **stuck for good**?" |
| On failure | Removed from endpoints | Container **restarted** |
| Runs | Always | Always |
| Recovers without restart | **Yes** | n/a (it restarts) |
| Typical check | Can I reach my dependencies and finish warm-up? | Is my process responsive? |

## Good readiness checks

Make the probe reflect "I can do useful work":

- **Warm-up done**: caches loaded, migrations finished, connections established.
- **Local dependencies OK**: the database connection pool is usable.
- **Not overloaded or draining**: an app can deliberately report not ready while shutting down.

### Node.js and NestJS example

```ts
// NestJS with @nestjs/terminus
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
  ) {}

  @Get('live')                       // used by the liveness probe: no dependencies
  live() { return this.health.check([]); }

  @Get('ready')                      // used by the readiness probe: can I serve requests?
  ready() { return this.health.check([() => this.db.pingCheck('database')]); }
}
```

### Be careful with shared dependencies

If **every** Pod checks the same database in readiness and the database blips, **all Pods become NotReady at once** and the whole service goes down, even if the app could have degraded gracefully. Check what **this Pod** needs locally, avoid deep checks on shared systems, and use sensible thresholds.

## Graceful shutdown

When a Pod is deleted, it is removed from endpoints **at the same time** as the container receives SIGTERM. Load balancers and kube-proxy take a moment to notice, so requests can still arrive for a few seconds. The usual fix:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]     # keep serving while endpoints propagate
terminationGracePeriodSeconds: 30
```

Your app should also stop accepting new connections on SIGTERM and finish in-flight requests.

## Other things worth knowing

- **`minReadySeconds`** (on a Deployment): a Pod must stay Ready this long to count as available, which slows a rollout so bad Pods are caught.
- **`publishNotReadyAddresses: true`** (on a Service): publish DNS for Pods even when not Ready. Used by StatefulSets that must find each other during startup.
- **Readiness gates** (`spec.readinessGates`): extra conditions set by controllers, for example the AWS Load Balancer Controller waiting for a load balancer target to be healthy.

## Lab

```bash
cd labs/07-autoscaling/manifests
```

**1. Pods start NotReady, the Service has no endpoints**

```bash
kubectl apply -f readiness-deploy.yaml
kubectl get pods -l app=readiness-demo                  # READY 0/1, because /ready.html does not exist
kubectl get endpointslices -l kubernetes.io/service-name=readiness-svc    # no endpoints
kubectl describe pod -l app=readiness-demo | grep -m1 "Readiness probe failed"
```

**2. Make them ready**

```bash
for p in $(kubectl get pods -l app=readiness-demo -o name); do
  kubectl exec $p -- sh -c 'echo ok > /usr/share/nginx/html/ready.html'
done
kubectl get pods -l app=readiness-demo                  # 1/1
kubectl get endpointslices -l kubernetes.io/service-name=readiness-svc -o wide     # 3 endpoints
```

**3. One Pod goes NotReady: removed from traffic, not restarted**

```bash
POD=$(kubectl get pod -l app=readiness-demo -o name | head -1)
kubectl exec $POD -- rm /usr/share/nginx/html/ready.html
kubectl get pods -l app=readiness-demo                  # one Pod 0/1, RESTARTS still 0
kubectl get endpointslices -l kubernetes.io/service-name=readiness-svc -o wide     # 2 endpoints
kubectl exec $POD -- sh -c 'echo ok > /usr/share/nginx/html/ready.html'
kubectl get pods -l app=readiness-demo                  # back to 3 ready
```

**4. Readiness protects a rolling update**

```bash
kubectl set image deployment/readiness-demo nginx=nginx:1.26     # the new Pods have no ready.html
kubectl rollout status deployment/readiness-demo --timeout=30s   # waits, does not finish
kubectl get pods -l app=readiness-demo                           # old Pods still serving, the new one is 0/1
kubectl rollout undo deployment/readiness-demo
```

**Cleanup**

```bash
kubectl delete -f readiness-deploy.yaml
```

## Gotchas

- **No readiness probe = Ready immediately**, so traffic hits Pods that are still starting.
- A readiness failure **never restarts** the container, so a permanently failing probe leaves a Pod running but unused.
- A readiness path that needs authentication, or a wrong port, makes every Pod permanently NotReady.
- Probes need `timeoutSeconds` that fit your slowest healthy response. The default 1 second is strict.
- A deep check on a shared dependency can take **all** Pods out of rotation together.
- During shutdown, add a `preStop` delay or you may drop requests on every deploy.

## Check yourself

1. What happens to a Pod when its readiness probe fails?
2. Does a failing readiness probe restart the container?
3. Which HTTP status codes count as success?
4. How does readiness protect a rolling update?
5. Why can a deep dependency check in readiness be dangerous?
6. What is a Pod's readiness if you define no probe?

<details>
<summary>Answers</summary>

1. It becomes NotReady and is removed from Service endpoints, so it gets no new traffic
2. No. Only a liveness (or startup) probe failure restarts it
3. 200 to 399
4. The Deployment only removes old Pods after the new ones are Ready, so bad versions never take over
5. If a shared dependency blips, every Pod becomes NotReady at once and the whole service goes down
6. Ready as soon as the container starts

</details>

---

[README](README.md) | [Liveness Probe →](liveness-probe.md)