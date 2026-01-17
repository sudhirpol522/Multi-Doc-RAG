# Quick Install Prometheus + Grafana on Minikube

## Prerequisites
- Minikube running
- Helm installed

## Install Helm (if not installed)
```powershell
# Windows (PowerShell)
choco install kubernetes-helm

# Or download from: https://github.com/helm/helm/releases
```

## Install Monitoring Stack
```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install (this takes 2-3 minutes)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Check installation
kubectl get pods -n monitoring
```

## Access Grafana
```bash
# Port-forward
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Open: http://localhost:3000
# Login: admin / admin123
```

## Access Prometheus
```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Open: http://localhost:9090
```

## What's Included
- Prometheus (metrics collection)
- Grafana (visualization)
- Alertmanager (alerting)
- Node Exporter (node metrics)
- Kube State Metrics (k8s object metrics)
- Pre-configured dashboards for Kubernetes

## Pre-built Dashboards in Grafana
After logging in, check:
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace (Pods)
- Kubernetes / Compute Resources / Pod
- Node Exporter / Nodes
