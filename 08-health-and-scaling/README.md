# Lab: Health and Scaling

> Status: 📘 Notes ready

## Goal

Make an app **observable** to Kubernetes (probes), then let the cluster **scale it** (HPA) and **size it** (VPA).

```
probes   ─►  Kubernetes knows if each Pod is started, alive and ready
HPA      ─►  more Pods when CPU rises, fewer when it falls
VPA      ─►  better CPU and memory requests for each Pod
```

## Setup

```bash
minikube start --driver=docker
minikube addons enable metrics-server        # CPU/memory metrics for kubectl top and the HPA
kubectl top nodes                            # works after about a minute
cd labs/07-autoscaling/manifests
```

- **kind**: install metrics-server from its official release manifest, then add `--kubelet-insecure-tls` to its container args.
- **VPA** (only for the VPA lab) is installed separately, see [the VPA note](../../08-health-and-scaling/vertical-pod-autoscaler.md#lab).

## Topic labs

| Note | What you do |
|---|---|
| [Readiness probe](../../08-health-and-scaling/readiness-probe.md) | Pods start NotReady, join and leave the endpoints, protect a rollout |
| [Liveness probe](../../08-health-and-scaling/liveness-probe.md) | Restart a hung container (exec and HTTP) |
| [Startup probe](../../08-health-and-scaling/startup-probe.md) | A slow app killed without a startup probe, saved with one |
| [HPA](../../08-health-and-scaling/horizontal-pod-autoscaler.md) | Load test, scale up, slow scale down, `behavior` |
| [VPA](../../08-health-and-scaling/vertical-pod-autoscaler.md) | Recommendations, `Recreate`, in-place resize |

## Combined scenario: ship an app safely

`php-apache.yaml` is an app with **requests**, a **readiness probe** and a **liveness probe**, plus an HPA that targets it.

### 1. Deploy

```bash
kubectl apply -f php-apache.yaml
kubectl apply -f hpa.yaml
kubectl get deploy,pods,hpa -l 'app in (php-apache)'
kubectl get hpa php-apache                              # <unknown> for a moment, then 0%/50%
kubectl describe pod -l app=php-apache | grep -A6 -E "Liveness|Readiness|Requests"
```

### 2. Load test, with the Service watching readiness

```bash
# terminal 2:  kubectl get hpa php-apache -w
# terminal 3:  kubectl get pods -l app=php-apache -w
kubectl apply -f load-generator.yaml
```

Questions to answer from what you see:

1. Why do new Pods show `0/1` for a few seconds? Which probe causes it?
2. Did traffic reach a new Pod **before** it was Ready? Check `kubectl get endpointslices -l kubernetes.io/service-name=php-apache`.
3. Use the formula to explain the first replica jump (current replicas × current % / 50).

### 3. Break the app during a rollout

```bash
kubectl set image deployment/php-apache php-apache=registry.k8s.io/does-not-exist:1.0
kubectl rollout status deployment/php-apache --timeout=30s     # does not complete
kubectl get pods -l app=php-apache                              # old Pods keep serving, new ones fail to start
kubectl rollout undo deployment/php-apache
```

The readiness probe and the rollout strategy kept the service up while the bad version could not become Ready.

### 4. Scale down

```bash
kubectl delete pod load-generator
kubectl get hpa php-apache -w                           # shrinks after about 5 minutes
kubectl apply -f hpa-behavior.yaml                      # shorter window: repeat to see it shrink faster
```

### 5. Right-size with VPA (optional)

Install VPA (see the VPA note), apply `vpa-hamster.yaml`, wait about five minutes, and compare the recommendation with the requests you guessed.

## Cleanup

```bash
kubectl delete -f hpa.yaml -f php-apache.yaml
kubectl delete pod load-generator --ignore-not-found
kubectl delete -f vpa-hamster.yaml --ignore-not-found
```

## What to take away

- **Readiness decides who gets traffic.** Liveness decides who gets restarted. Startup protects slow boots.
- **Requests are the denominator** for HPA utilization, so set them first.
- Scale **up fast, down slowly**. Scale-down waits five minutes by default.
- HPA adds Pods. **Node capacity** still has to exist (or be added by a cluster autoscaler).
- Start VPA in `Off` mode and use it to choose sensible requests.

## Check yourself

1. Why did the bad image rollout not take the app down?
2. Why does the HPA need CPU requests?
3. What stops new Pods from receiving traffic before they can serve it?

<details>
<summary>Answers</summary>

1. New Pods never became Ready, so the Deployment kept the old Pods and the Service only routed to Ready Pods
2. Utilization is measured as a percentage of the request, so without one there is nothing to compare to
3. The readiness probe: a Pod is added to the Service endpoints only once it is Ready

</details>