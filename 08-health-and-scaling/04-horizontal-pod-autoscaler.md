# Horizontal Pod Autoscaler (HPA)

> Status: 📘 Notes ready
>
> YAML files for the labs: [`labs/07-autoscaling/manifests/`](../labs/07-autoscaling/manifests/) · Lab: [`labs/07-autoscaling`](../labs/07-autoscaling/)

[← Startup Probe](startup-probe.md) | [README](README.md) | [Vertical Pod Autoscaler →](vertical-pod-autoscaler.md)

---

## Idea

The **Horizontal Pod Autoscaler** changes the **number of Pod replicas** of a Deployment (or StatefulSet) to match load. More traffic means more Pods, less traffic means fewer.

```
load ↑  →  CPU per Pod ↑  →  HPA raises replicas  →  load spreads  →  CPU per Pod ↓
load ↓  →  CPU per Pod ↓  →  HPA lowers replicas  (after a cool-down)
```

| Kind of scaling | Changes | Tool |
|---|---|---|
| **Horizontal** | Number of Pods | **HPA** |
| Vertical | Size (requests) of each Pod | [VPA](vertical-pod-autoscaler.md) |
| Cluster | Number of nodes | Cluster Autoscaler, Karpenter |

## Requirements

| Needed | Why |
|---|---|
| **metrics-server** installed | Provides CPU and memory metrics (`kubectl top` uses it) |
| **CPU requests** on the containers | Utilization is measured as a **percentage of the request** |
| A workload with a **scale subresource** | Deployments, StatefulSets, ReplicaSets |
| Pods that can run **several copies** | Stateless, or at least safe to replicate |

```bash
kubectl top nodes                                   # works only if metrics-server is healthy
kubectl get apiservice v1beta1.metrics.k8s.io       # Available: True
```

Install on minikube with `minikube addons enable metrics-server`. On kind, apply the official `components.yaml` and add `--kubelet-insecure-tls` to its args.

## YAML explained

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:                  # WHAT to scale
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1                   # never fewer than this
  maxReplicas: 10                  # never more than this
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization        # percent of the CPU REQUEST
          averageUtilization: 50   # keep the average at 50%
```

Quick imperative form (flag names vary a little by kubectl version, see `kubectl autoscale --help`):

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

## The formula

```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / targetMetricValue ) )
```

| Situation | Calculation | Result |
|---|---|---|
| 3 replicas at 80% CPU, target 50% | ceil(3 × 80 / 50) = ceil(4.8) | **5** replicas |
| 5 replicas at 20% CPU, target 50% | ceil(5 × 20 / 50) = ceil(2.0) | **2** replicas |
| 4 replicas at 52% CPU, target 50% | ratio 1.04 is within the **10% tolerance** | **no change** |

- The controller checks about every **15 seconds**.
- Changes smaller than the **tolerance (10% by default)** are ignored to prevent flapping.
- The result is clamped to `minReplicas` and `maxReplicas`.

## What it can scale on

| Metric `type` | Meaning | Needs |
|---|---|---|
| **Resource** | CPU or memory of Pods (`Utilization` percent, or `AverageValue`) | metrics-server |
| **ContainerResource** | CPU or memory of **one container** in a Pod (ignores sidecars) | metrics-server |
| **Pods** | A custom per-Pod metric, such as requests per second | A custom-metrics adapter (for example Prometheus Adapter) |
| **Object** | A metric of another object, such as an Ingress | A custom-metrics adapter |
| **External** | A metric from outside the cluster, such as queue length | An external-metrics adapter |

With several metrics, the HPA computes a desired count for **each** and uses the **highest**.

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 60 }
  - type: Resource
    resource:
      name: memory
      target: { type: AverageValue, averageValue: 400Mi }
```

**Event-driven scaling** (queue depth, Kafka lag, cron schedules, scale to zero) is usually done with **KEDA**, which creates and manages HPAs for you.

## Controlling speed: `behavior`

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0        # react immediately
      selectPolicy: Max                    # use the policy that allows the biggest change
      policies:
        - type: Pods
          value: 4
          periodSeconds: 15                # add at most 4 Pods per 15s
        - type: Percent
          value: 100
          periodSeconds: 15                # or double every 15s
    scaleDown:
      stabilizationWindowSeconds: 300      # wait 5 minutes of lower demand before shrinking
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60                # remove at most half per minute
```

| Setting | Meaning | Default |
|---|---|---|
| `stabilizationWindowSeconds` | Look back this long and use the **safest** value, to avoid flapping | scale up: 0, scale down: **300** |
| `policies` | The maximum rate of change | up: 100% or 4 Pods per 15s, down: 100% per 15s |
| `selectPolicy` | `Max`, `Min` or `Disabled` (turns that direction off) | `Max` |

Scale **up fast, scale down slowly** is the safe pattern.

## Reading its status

```bash
kubectl get hpa
kubectl describe hpa php-apache
```

```
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS
php-apache   Deployment/php-apache   68%/50%    1         10        3
```

| `TARGETS` | Meaning |
|---|---|
| `68%/50%` | Current average / target |
| `<unknown>/50%` | No metrics: metrics-server missing, or no CPU **requests** set, or the Pods are too new |

`describe` shows the **conditions**: `AbleToScale`, `ScalingActive` (metrics are available), `ScalingLimited` (hit min or max).

## HPA and the rest of the system

- **Readiness probes** matter: a new Pod only helps once it is Ready, and its early metrics are treated conservatively. Slow startup delays the benefit of scaling up (see [startup probe](startup-probe.md)).
- **Requests** are the denominator. Set them from real measurements, see [requests and limits](../07-scheduling/resource-requests-limits.md).
- **Cluster capacity**: new Pods that do not fit stay `Pending`. The **Cluster Autoscaler** or **Karpenter** reads those Pending Pods and adds nodes.
- **Do not set `replicas` in the Deployment YAML** when an HPA manages it, or every `kubectl apply` resets the replica count. Omit the field.
- **Do not combine** an HPA on CPU/memory with a **VPA** that changes the same requests on the same workload.

## Lab

```bash
minikube addons enable metrics-server        # kind: install metrics-server and add --kubelet-insecure-tls
kubectl top nodes                            # works after about a minute
cd labs/07-autoscaling/manifests
```

**1. Deploy the app and the HPA**

```bash
kubectl apply -f php-apache.yaml             # Deployment (200m request) with probes, plus a Service
kubectl apply -f hpa.yaml                    # target 50% CPU, 1 to 10 replicas
kubectl get hpa php-apache                   # TARGETS <unknown>/50% for up to a minute, then 0%/50%
```

**2. Generate load** (use a second terminal for the watches)

```bash
# terminal 2
kubectl get hpa php-apache -w
# terminal 3
kubectl get pods -l app=php-apache -w
# terminal 1
kubectl apply -f load-generator.yaml
```

Watch the TARGETS climb well above 50%, REPLICAS rise (1 → 4 → 8 → ...), and new Pods go `Running` and then Ready.

```bash
kubectl describe hpa php-apache | tail -15            # SuccessfulRescale events and the reason
kubectl top pods -l app=php-apache
```

**3. Remove the load: scale down is slow by design**

```bash
kubectl delete pod load-generator
kubectl get hpa php-apache -w                         # utilization drops, but replicas stay for about 5 minutes
```

**4. Make scaling faster with `behavior`**

```bash
kubectl apply -f hpa-behavior.yaml                    # scale-down window 60s, at most 50% per 30s
kubectl apply -f load-generator.yaml                  # load again
# wait for scale-up, then:
kubectl delete pod load-generator
kubectl get hpa php-apache -w                         # replicas now shrink after about a minute
```

**5. The HPA wins against manual changes**

```bash
kubectl scale deployment php-apache --replicas=6
kubectl get deploy php-apache -w                      # the HPA brings it back to the value it computes
```

**6. Break it on purpose**

```bash
kubectl patch deployment php-apache --type=json -p='[{"op":"remove","path":"/spec/template/spec/containers/0/resources/requests"}]'
kubectl describe hpa php-apache | grep -i -A2 "unable\|missing"   # failed to get cpu utilization: missing request
kubectl apply -f php-apache.yaml                      # restore the requests
```

**Cleanup**

```bash
kubectl delete -f hpa.yaml -f php-apache.yaml
kubectl delete pod load-generator --ignore-not-found
```

## Gotchas

- **`<unknown>` targets**: install metrics-server and set CPU **requests**.
- **`replicas` in your manifest** fights the HPA. Remove it.
- **Scale-down looks "stuck"**: it waits 5 minutes by default (stabilization window).
- A **memory** HPA often does not work: many apps do not release memory when load drops, so the metric never falls.
- **Limits are not the denominator.** Utilization is relative to **requests**, so values above 100% are normal.
- **Slow startup** wastes the scale-up. Use probes and keep images small.
- The HPA only adds Pods. If nodes are full, you need a cluster autoscaler.
- Stateful or non-replicable workloads need care before enabling an HPA.

## Check yourself

1. What does the HPA change, and what does it measure for CPU?
2. Compute the desired replicas: 4 replicas at 90% CPU with a target of 60%.
3. Why does a Deployment without CPU requests show `<unknown>`?
4. How long does scale-down wait by default, and how do you change it?
5. What do you do with `replicas:` in the Deployment manifest when an HPA manages it?
6. Which tool handles scaling on queue length or scale to zero?
7. What adds nodes when scaled-up Pods have no room?

<details>
<summary>Answers</summary>

1. The number of replicas. It measures CPU usage as a percentage of the Pods' CPU **requests**
2. ceil(4 × 90 / 60) = 6
3. Utilization is a percentage of the request, so without a request there is nothing to compare
4. About 300 seconds (the stabilization window). Change `behavior.scaleDown.stabilizationWindowSeconds`
5. Remove it, so `kubectl apply` does not reset the count
6. KEDA
7. The Cluster Autoscaler or Karpenter

</details>

---

[← Startup Probe](startup-probe.md) | [README](README.md) | [Vertical Pod Autoscaler →](vertical-pod-autoscaler.md)