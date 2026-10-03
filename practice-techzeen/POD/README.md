run pod 

```bash
kubctl apply -f pod.yaml
```

get pod

```bash
kubectl get pod
```

```bash
kubectl describe pod pizza
```


```bash
kubectl create deployment pizza-deployment --image=kicbase/echo-server:1.0
```

```bash
kubectl scale deployment pizza-deployment --replicas=3
```

```bash
kubectl get deployment pizza-deployment
```

```bash
kubectl get pods
```

```bash
kubectl get pods -o wide
```

```bash
kubectl get pods --selector=department=pizza
```

```bash
kubectl delete deployment pizza-deployment
```

```bash
kubectl delete pod pizza
```
