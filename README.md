# k8s-monitoring — Kubernetes Observability & Alerting Stack

![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)
![Alertmanager](https://img.shields.io/badge/Alertmanager-E6522C?logo=prometheus&logoColor=white)
![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-orange)
![Kubernetes](https://img.shields.io/badge/Kubernetes-KIND-blue)

A complete Kubernetes observability stack deployed via GitOps using ArgoCD.
Covers metrics collection, dashboards, alerting, and email notifications for a real microservice running in dev and staging environments.

> **GitOps Config Repo:** [gitops-config](https://github.com/kiran-kumatkar/gitops-config)
> **Application Repo:** [sample-app](https://github.com/kiran-kumatkar/sample-app)

---

## Stack Components

| Component | Purpose |
|-----------|---------|
| **Prometheus Operator** | Manages Prometheus and Alertmanager as Kubernetes CRDs |
| **Prometheus** | Scrapes and stores metrics from cluster and applications |
| **kube-state-metrics** | Exposes Kubernetes object state (pod status, replica counts, etc.) |
| **node-exporter** | Exposes host-level metrics (CPU, memory, disk per node) |
| **Grafana** | Dashboards and visualization |
| **Alertmanager** | Alert routing and email notifications |

---

## Architecture

```
Kubernetes Cluster (KIND)
│
├── kube-state-metrics     → Kubernetes object metrics (deployments, pods, replicas)
├── node-exporter          → Node level metrics (CPU, memory, disk, network)
│
├── Prometheus             → Scrapes all targets via ServiceMonitor CRDs
│   ├── ServiceMonitor     → Tells Prometheus what to scrape (sample-app in dev/staging)
│   └── PrometheusRule     → Custom alert rules for sample-app scenarios
│
├── Alertmanager           → Receives firing alerts from Prometheus
│   └── email-alerts       → Routes alerts to Gmail via SMTP
│
└── Grafana                → Pre-loaded dashboards from grafana.com
    ├── Kubernetes Cluster Overview  (ID: 7249)
    ├── Node Exporter Full           (ID: 1860)
    ├── ArgoCD Dashboard             (ID: 14584)
    └── Pod Resource Usage           (ID: 6417)
```

---

## Repository Structure

```
k8s-monitoring/
├── helm/
│   └── values/
│       └── kube-prometheus-stack-values.yaml   # Full stack configuration
│── sample-app-alerts.yaml                  # Custom PrometheusRules
│── sample-app-monitor.yaml                 # ServiceMonitor for sample-app
└── README.md
```

---

## How ArgoCD Deploys This

This repo is managed by two ArgoCD Applications defined in
[gitops-config/apps/](https://github.com/kiran-kumatkar/gitops-config/tree/main/apps):

| ArgoCD App | Source | What it deploys |
|------------|--------|----------------|
| `kube-prometheus-stack` | `helm/values/` + Prometheus Community chart | Full monitoring stack |
| `monitoring-extras` | `rules/` + `servicemonitor/` | Custom rules and monitors |

Any change pushed to this repo is automatically detected and synced by ArgoCD.

---

## Custom Alert Rules

Defined in `rules/sample-app-alerts.yaml` — covers real-world failure scenarios:

| Alert | Trigger | Severity |
|-------|---------|----------|
| `SampleAppCrashLooping` | Pod restarting repeatedly | Critical |
| `SampleAppDown` | 0 available replicas | Critical |
| `SampleAppReplicasMismatch` | Desired != available replicas | Warning |
| `SampleAppPodPending` | Pod stuck Pending for 5 minutes | Warning |
| `SampleAppImagePullFailed` | ImagePullBackOff or ErrImagePull | Critical |

All alerts route to email via Alertmanager. Resolved alerts also send an email.

---

## Real World Scenarios Simulated

### Scenario 1 — CrashLoopBackOff
```bash
# Deploy bad image — app exits immediately on startup
kubectl set image deployment/sample-app \
  sample-app=kirankumatkar217/sample-app:bad -n dev

# Alert fires: SampleAppCrashLooping
# Debug: kubectl logs deployment/sample-app -n dev --previous
# Fix: revert image in gitops-config repo and push
```

### Scenario 2 — App Completely Down
```bash
# Scale to 0 replicas
kubectl scale deployment sample-app --replicas=0 -n dev

# Alert fires: SampleAppDown
# Debug: kubectl get events -n dev --sort-by='.lastTimestamp'
# Fix: kubectl scale deployment sample-app --replicas=3 -n dev
```

### Scenario 3 — Image Pull Failure
```bash
# Set non-existent image tag
kubectl set image deployment/sample-app \
  sample-app=kirankumatkar217/sample-app:doesnotexist -n dev

# Alert fires: SampleAppImagePullFailed
# Debug: kubectl describe pod <pod> -n dev
# Fix: correct image tag in gitops-config and push
```

### Scenario 4 — Replica Mismatch
```bash
# Request more replicas than cluster can fit
kubectl scale deployment sample-app --replicas=20 -n dev

# Alert fires: SampleAppReplicasMismatch + SampleAppPodPending
# Debug: kubectl describe nodes | grep -A5 "Allocated resources"
# Fix: reduce replicas to fit available node resources
```

---

## Access UIs

```bash
# Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# http://localhost:3000  |  admin / <your-password>

# Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# http://localhost:9090

# Alertmanager
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
# http://localhost:9093
```

---

## Useful Prometheus Queries

```promql
# Pod restart rate (detects crash loops)
rate(kube_pod_container_status_restarts_total{namespace=~"dev|staging"}[5m]) * 60

# Available replicas for sample-app
kube_deployment_status_replicas_available{deployment="sample-app"}

# Memory usage as % of limit
(container_memory_working_set_bytes{container="sample-app"}
  / kube_pod_container_resource_limits{container="sample-app", resource="memory"}) * 100

# CPU usage
rate(container_cpu_usage_seconds_total{container="sample-app"}[5m])

# Pods not ready
kube_pod_status_ready{condition="true", namespace=~"dev|staging"} == 0
```

---

## Useful Debug Commands

```bash
# Check all monitoring pods are healthy
kubectl get pods -n monitoring

# Check Prometheus targets are being scraped
# http://localhost:9090/targets

# Check alert rules loaded in Prometheus
# http://localhost:9090/rules

# Check Alertmanager config loaded correctly
# http://localhost:9093/#/status

# View firing alerts
kubectl get prometheusrule -n monitoring

# Check ServiceMonitors
kubectl get servicemonitor -n monitoring

# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -n dev
kubectl top pods -n staging

# Recent cluster events
kubectl get events -n dev --sort-by='.lastTimestamp'
```

---

## Prerequisites — Manual Step Before Deploying

Create the SMTP secret manually in the cluster (not stored in Git):

```bash
# Gmail App Password — get from:
# Google Account → Security → 2-Step Verification → App Passwords
kubectl create secret generic alertmanager-email-secret \
  --from-literal=smtp_password='your-16-char-app-password' \
  --namespace monitoring
```

---

## Bootstrap

This stack is deployed via ArgoCD from
[gitops-config](https://github.com/kiran-kumatkar/gitops-config).
Once ArgoCD is running, just push changes here and they deploy automatically.

```bash
# Verify monitoring stack is synced
argocd app get kube-prometheus-stack
argocd app get monitoring-extras

# Force sync if needed
argocd app sync kube-prometheus-stack
argocd app sync monitoring-extras
```

---

# Screenshots

All command outputs and observations are captured in the [screenshots/](./screenshots) folder.
---
