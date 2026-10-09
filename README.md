# Kubernetes Learning Notes

This repository is a structured learning path for Kubernetes, covering prerequisites, core concepts, workloads, networking, security, storage, troubleshooting, Helm, observability, and production practices.

## Table of Contents

- [Prerequisites](./00-prerequisites/README.md)
  - [Linux](./00-prerequisites/01-linux/README.md)
  - [Networking](./00-prerequisites/02-networking/README.md)
  - [Docker](./00-prerequisites/03-docker/README.md)
  - [YAML](./00-prerequisites/04-yaml/README.md)

- [Kubernetes Basics](./01-kubernetes-basics/00-what-is-kubernetes.md)
  - [Why Kubernetes](./01-kubernetes-basics/01-why-kubernetes.md)
  - [Architecture](./01-kubernetes-basics/02-architecture.md)
  - [Cluster](./01-kubernetes-basics/03-cluster.md)
  - [Nodes](./01-kubernetes-basics/04-nodes.md)
  - [kubectl](./01-kubernetes-basics/05-kubectl.md)
  - [Namespaces](./01-kubernetes-basics/06-namespace.md)

- [Kubernetes Architecture](./02-kubernetes-architecture/README.md)
  - [Control Plane](./02-kubernetes-architecture/01-control-plane/)
  - [Worker Node](./02-kubernetes-architecture/02-worker-node/)
  - [Request Flow](./02-kubernetes-architecture/03-request-flow.md)

- [Kubernetes Workloads](./03-kubernetes-workloads/README.md)
  - [Pods](./03-kubernetes-workloads/01-pods/README.md)
  - [Deployments](./03-kubernetes-workloads/02-deployments/README.md)
  - [ReplicaSets](./03-kubernetes-workloads/03-replicasets/README.md)
  - [DaemonSets](./03-kubernetes-workloads/04-daemonsets/README.md)
  - [StatefulSets](./03-kubernetes-workloads/05-statefulsets/README.md)
  - [Jobs](./03-kubernetes-workloads/06-jobs/README.md)
  - [CronJobs](./03-kubernetes-workloads/07-cronjobs/README.md)

- [Kubernetes Configuration](./04-kubernetes-configuration/README.md)
  - [ConfigMaps](./04-kubernetes-configuration/01-configmaps/README.md)
  - [Secrets](./04-kubernetes-configuration/02-secrets/README.md)
  - [Environment Variables](./04-kubernetes-configuration/03-environment-variables/README.md)
  - [YAML Manifests](./04-kubernetes-configuration/04-yaml-manifests/README.md)

- [Kubernetes Networking](./05-kubernetes-networking/README.md)
  - [Pod Networking](./05-kubernetes-networking/01-pod-networking.md)
  - [Services](./05-kubernetes-networking/02-services/)
  - [DNS](./05-kubernetes-networking/03-dns.md)
  - [Ingress](./05-kubernetes-networking/04-ingress/)
  - [Network Policies](./05-kubernetes-networking/05-network-policies/)

- [Storage](./06-storage/README.md)
  - [Volumes](./06-storage/volumes.md)
  - [EmptyDir](./06-storage/emptydir.md)
  - [HostPath](./06-storage/hostpath.md)
  - [Persistent Volumes](./06-storage/persistent-volumes.md)
  - [Persistent Volume Claims](./06-storage/persistent-volume-claims.md)
  - [Storage Classes](./06-storage/storage-classes.md)

- [Scheduling](./07-scheduling/README.md)
  - [Labels & Selectors](./07-scheduling/labels-selectors.md)
  - [Node Selector](./07-scheduling/node-selector.md)
  - [Node Affinity](./07-scheduling/node-affinity.md)
  - [Pod Affinity](./07-scheduling/pod-affinity.md)
  - [Taints & Tolerations](./07-scheduling/taints-tolerations.md)
  - [Resource Requests & Limits](./07-scheduling/resource-requests-limits.md)

- [Health and Scaling](./08-health-and-scaling/README.md)
  - [Liveness Probe](./08-health-and-scaling/liveness-probe.md)
  - [Readiness Probe](./08-health-and-scaling/readiness-probe.md)
  - [Startup Probe](./08-health-and-scaling/startup-probe.md)
  - [Horizontal Pod Autoscaler](./08-health-and-scaling/horizontal-pod-autoscaler.md)
  - [Vertical Pod Autoscaler](./08-health-and-scaling/vertical-pod-autoscaler.md)

- [Security](./09-security/README.md)
  - [Service Accounts](./09-security/01-service-accounts.md)
  - [RBAC](./09-security/02-RBAC/README.md)
  - [Security Context](./09-security/03-security-context.md)
  - [Pod Security](./09-security/04-pod-security.md)
  - [Network Security](./09-security/05-network-security.md)

- [Helm](./10-helm/README.md)
  - [Helm Basics](./10-helm/helm-basics.md)
  - [Charts](./10-helm/charts.md)
  - [Templates](./10-helm/templates.md)
  - [Values](./10-helm/values.md)
  - [My First Chart](./10-helm/05-my-first-chart/README.md)

- [Troubleshooting](./11-troubleshooting/README.md)
  - [Pod Not Starting](./11-troubleshooting/pod-not-starting.md)
  - [CrashLoopBackOff](./11-troubleshooting/crashloopbackoff.md)
  - [ImagePullBackOff](./11-troubleshooting/imagepullbackoff.md)
  - [Pending Pods](./11-troubleshooting/pending-pods.md)
  - [Service Not Working](./11-troubleshooting/service-not-working.md)
  - [DNS Issues](./11-troubleshooting/dns-issues.md)
  - [Debugging Commands](./11-troubleshooting/debugging-commands.md)

- [Kubernetes Operations](./12-kubernetes-operations/README.md)
  - [Rolling Updates](./12-kubernetes-operations/rolling-updates.md)
  - [Rollbacks](./12-kubernetes-operations/rollbacks.md)
  - [Upgrades](./12-kubernetes-operations/upgrades.md)
  - [Backups](./12-kubernetes-operations/backups.md)
  - [Cluster Maintenance](./12-kubernetes-operations/cluster-maintenance.md)

- [Observability](./13-observability/README.md)
  - [Logging](./13-observability/logging.md)
  - [Metrics](./13-observability/metrics.md)
  - [Monitoring](./13-observability/monitoring.md)
  - [Alerting](./13-observability/alerting.md)

- [Kubernetes Production](./14-kubernetes-production/README.md)
  - [High Availability](./14-kubernetes-production/high-availability.md)
  - [Cluster Design](./14-kubernetes-production/cluster-design.md)
  - [Resource Management](./14-kubernetes-production/resource-management.md)
  - [Security Hardening](./14-kubernetes-production/security-hardening.md)
  - [Production Checklist](./14-kubernetes-production/production-checklist.md)

- [AWS EKS](./15-aws-eks/README.md)
  - [EKS Basics](./15-aws-eks/eks-basics.md)
  - [EKS Cluster](./15-aws-eks/eks-cluster.md)
  - [Node Groups](./15-aws-eks/node-groups.md)
  - [IAM](./15-aws-eks/iam.md)
  - [Load Balancer Controller](./15-aws-eks/load-balancer-controller.md)
  - [EBS CSI](./15-aws-eks/ebs-csi.md)
  - [IRSA](./15-aws-eks/irsa.md)

- [Advanced Topics](./16-advanced/README.md)
  - [Operators](./16-advanced/operators.md)
  - [Custom Resources](./16-advanced/custom-resources.md)
  - [CRD](./16-advanced/CRD.md)
  - [Admission Controllers](./16-advanced/admission-controllers.md)
  - [Service Mesh](./16-advanced/service-mesh.md)

- [Labs](./17-labs/01-first-pod/README.md)
  - [Deployment Lab](./17-labs/02-deployment/README.md)
  - [Service Lab](./17-labs/03-service/README.md)
  - [ConfigMap & Secret Lab](./17-labs/04-configmap-secret/README.md)
  - [Ingress Lab](./17-labs/05-ingress/README.md)
  - [Storage Lab](./17-labs/06-storage/README.md)
  - [Autoscaling Lab](./17-labs/07-autoscaling/README.md)
  - [RBAC Lab](./17-labs/08-rbac/README.md)
  - [Helm Lab](./17-labs/09-helm/README.md)
  - [Production Project Lab](./17-labs/10-production-project/README.md)

- [Projects](./18-projects/01-nodejs-kubernetes/README.md)
  - [Fullstack Kubernetes](./18-projects/02-fullstack-kubernetes/README.md)
  - [Monitoring Stack](./18-projects/03-monitoring-stack/README.md)
  - [Production EKS](./18-projects/04-production-eks/README.md)

- [Cheat Sheets](./19-cheatsheets/)
  - [kubectl](./19-cheatsheets/01-kubectl.md)
  - [YAML](./19-cheatsheets/02-yaml.md)
  - [Troubleshooting](./19-cheatsheets/03-troubleshooting.md)
  - [Kubernetes Commands](./19-cheatsheets/04-kubernetes-commands.md)

- [Learn from YouTube](./20-learn-from-youtube/practice-techzeen/)
  - [ConfigMap & Secrets](./20-learn-from-youtube/practice-techzeen/ConfigMap-Secrets/README.md)
  - [POD](./20-learn-from-youtube/practice-techzeen/POD/README.md)
  - [Service](./20-learn-from-youtube/practice-techzeen/Service/README.md)
  - [Helm](./20-learn-from-youtube/practice-techzeen/helm/README.md)
  - [React Project](./20-learn-from-youtube/practice-techzeen/react-project/README.md)
  - [Volume](./20-learn-from-youtube/practice-techzeen/volume/README.md)

## Repository Structure

```text
kubernetes-learning/
│
├── README.md
│
├── 00-prerequisites/
│   ├── README.md
│   ├── linux/
│   ├── networking/
│   ├── docker/
│   └── yaml/
│
├── 01-kubernetes-basics/
│   ├── README.md
│   ├── what-is-kubernetes.md
│   ├── why-kubernetes.md
│   ├── architecture.md
│   ├── cluster.md
│   ├── nodes.md
│   ├── kubectl.md
│   └── namespaces.md
│
├── 02-kubernetes-architecture/
│   ├── README.md
│   ├── control-plane/
│   │   ├── api-server.md
│   │   ├── etcd.md
│   │   ├── scheduler.md
│   │   └── controller-manager.md
│   │
│   ├── worker-node/
│   │   ├── kubelet.md
│   │   ├── kube-proxy.md
│   │   └── container-runtime.md
│   │
│   └── request-flow.md
│
├── 03-kubernetes-workloads/
│   ├── README.md
│   ├── pods/
│   ├── deployments/
│   ├── replicasets/
│   ├── daemonsets/
│   ├── statefulsets/
│   ├── jobs/
│   └── cronjobs/
│
├── 04-kubernetes-configuration/
│   ├── README.md
│   ├── configmaps/
│   ├── secrets/
│   ├── environment-variables/
│   └── yaml-manifests/
│
├── 05-kubernetes-networking/
│   ├── README.md
│   ├── pod-networking.md
│   ├── services/
│   │   ├── clusterip.md
│   │   ├── nodeport.md
│   │   └── loadbalancer.md
│   ├── dns.md
│   ├── ingress/
│   └── network-policies/
│
├── 06-storage/
│   ├── README.md
│   ├── volumes.md
│   ├── emptydir.md
│   ├── hostpath.md
│   ├── persistent-volumes.md
│   ├── persistent-volume-claims.md
│   └── storage-classes.md
│
├── 07-scheduling/
│   ├── README.md
│   ├── labels-selectors.md
│   ├── node-selector.md
│   ├── node-affinity.md
│   ├── pod-affinity.md
│   ├── taints-tolerations.md
│   └── resource-requests-limits.md
│
├── 08-health-and-scaling/
│   ├── README.md
│   ├── liveness-probe.md
│   ├── readiness-probe.md
│   ├── startup-probe.md
│   ├── horizontal-pod-autoscaler.md
│   └── vertical-pod-autoscaler.md
│
├── 09-security/
│   ├── README.md
│   ├── service-accounts.md
│   ├── RBAC/
│   ├── security-context.md
│   ├── pod-security.md
│   └── network-security.md
│
├── 10-helm/
│   ├── README.md
│   ├── helm-basics.md
│   ├── charts.md
│   ├── templates.md
│   ├── values.md
│   └── my-first-chart/
│
├── 11-troubleshooting/
│   ├── README.md
│   ├── pod-not-starting.md
│   ├── crashloopbackoff.md
│   ├── imagepullbackoff.md
│   ├── pending-pods.md
│   ├── service-not-working.md
│   ├── dns-issues.md
│   └── debugging-commands.md
│
├── 12-kubernetes-operations/
│   ├── README.md
│   ├── rolling-updates.md
│   ├── rollbacks.md
│   ├── upgrades.md
│   ├── backups.md
│   └── cluster-maintenance.md
│
├── 13-observability/
│   ├── README.md
│   ├── logging.md
│   ├── metrics.md
│   ├── monitoring.md
│   └── alerting.md
│
├── 14-kubernetes-production/
│   ├── README.md
│   ├── high-availability.md
│   ├── cluster-design.md
│   ├── resource-management.md
│   ├── security-hardening.md
│   └── production-checklist.md
│
├── 15-aws-eks/
│   ├── README.md
│   ├── eks-basics.md
│   ├── eks-cluster.md
│   ├── node-groups.md
│   ├── iam.md
│   ├── load-balancer-controller.md
│   ├── ebs-csi.md
│   └── irsa.md
│
├── 16-advanced/
│   ├── README.md
│   ├── operators.md
│   ├── custom-resources.md
│   ├── CRD.md
│   ├── admission-controllers.md
│   └── service-mesh.md
│
├── labs/
│   ├── 01-first-pod/
│   ├── 02-deployment/
│   ├── 03-service/
│   ├── 04-configmap-secret/
│   ├── 05-ingress/
│   ├── 06-storage/
│   ├── 07-autoscaling/
│   ├── 08-rbac/
│   ├── 09-helm/
│   └── 10-production-project/
│
├── projects/
│   ├── 01-nodejs-kubernetes/
│   ├── 02-fullstack-kubernetes/
│   ├── 03-monitoring-stack/
│   └── 04-production-eks/
│
└── cheatsheets/
    ├── kubectl.md
    ├── yaml.md
    ├── troubleshooting.md
    └── kubernetes-commands.md
```

This README is meant to serve as the main entry point for the learning journey and should be used as the landing page to navigate through the study modules and exercises.

