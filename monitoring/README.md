# Prometheus and Grafana Monitoring Setup

## Quick Install with Helm

### 1. Install Prometheus Stack (includes Grafana)
```bash
# Add Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (includes Prometheus, Grafana, and Alertmanager)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Wait for pods to be ready
kubectl wait --for=condition=Ready pods --all -n monitoring --timeout=300s
```

### 2. Access Grafana Dashboard
```bash
# Port-forward Grafana (run in separate terminal)
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Access at: http://localhost:3000
# Username: admin
# Password: admin123
```

### 3. Access Prometheus UI
```bash
# Port-forward Prometheus (run in separate terminal)
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Access at: http://localhost:9090
```

---

## Manual Installation (Alternative)

If you prefer manual installation without Helm:

### 1. Install Prometheus
```bash
kubectl apply -f monitoring/prometheus-namespace.yaml
kubectl apply -f monitoring/prometheus-deployment.yaml
kubectl apply -f monitoring/prometheus-service.yaml
kubectl apply -f monitoring/prometheus-configmap.yaml
kubectl apply -f monitoring/prometheus-rbac.yaml
```

### 2. Install Grafana
```bash
kubectl apply -f monitoring/grafana-deployment.yaml
kubectl apply -f monitoring/grafana-service.yaml
kubectl apply -f monitoring/grafana-configmap.yaml
```

---

## Configure ServiceMonitor for MultiDocChat

Create a ServiceMonitor to scrape metrics from your application:

```bash
kubectl apply -f monitoring/multi-doc-chat-servicemonitor.yaml
```

---

## Access Services

### Using Minikube Service
```bash
# Grafana
minikube service monitoring-grafana -n monitoring

# Prometheus
minikube service monitoring-kube-prometheus-prometheus -n monitoring
```

### Using Port-Forward
```bash
# Grafana (terminal 1)
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Prometheus (terminal 2)
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Alertmanager (terminal 3)
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

---

## Grafana Default Dashboards

After installation, Grafana comes with pre-configured dashboards:

1. **Kubernetes Cluster Monitoring**
2. **Kubernetes Pods**
3. **Kubernetes Nodes**
4. **Prometheus Stats**

---

## Add Custom Dashboard for MultiDocChat

1. Login to Grafana (http://localhost:3000)
2. Go to **Dashboards** → **Import**
3. Use the dashboard JSON from `monitoring/grafana-dashboard-multi-doc-chat.json`

---

## Metrics to Monitor

The setup will automatically collect:

- **Pod Metrics**: CPU, Memory, Network
- **Container Metrics**: Resource usage
- **Node Metrics**: Cluster health
- **API Server Metrics**: Request rates, latencies
- **Application Metrics**: Custom metrics from your app (if instrumented)

---

## Add Prometheus Instrumentation to FastAPI

To expose custom metrics from your FastAPI app, add to `main.py`:

```python
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Response

# Metrics
request_count = Counter('app_requests_total', 'Total app requests')
request_duration = Histogram('app_request_duration_seconds', 'Request duration')

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

# Add middleware to track metrics
@app.middleware("http")
async def track_metrics(request, call_next):
    request_count.inc()
    with request_duration.time():
        response = await call_next(request)
    return response
```

Then add to `requirements.txt`:
```
prometheus-client==0.20.0
```

---

## Useful Prometheus Queries

```promql
# CPU usage by pod
rate(container_cpu_usage_seconds_total{pod=~"multi-doc-chat.*"}[5m])

# Memory usage by pod
container_memory_usage_bytes{pod=~"multi-doc-chat.*"}

# Pod restart count
kube_pod_container_status_restarts_total{pod=~"multi-doc-chat.*"}

# HTTP request rate
rate(app_requests_total[5m])

# HTTP request duration
app_request_duration_seconds_sum / app_request_duration_seconds_count
```

---

## Alerting Rules

Create custom alerts in `monitoring/prometheus-rules.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: multi-doc-chat-alerts
  namespace: monitoring
spec:
  groups:
  - name: multi-doc-chat
    interval: 30s
    rules:
    - alert: PodDown
      expr: up{job="multi-doc-chat"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "MultiDocChat pod is down"
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes{pod=~"multi-doc-chat.*"} > 450000000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage in MultiDocChat pod"
```

---

## Cleanup

```bash
# Remove Prometheus stack
helm uninstall monitoring -n monitoring

# Or delete manually
kubectl delete namespace monitoring
```

---

## Troubleshooting

### Check Prometheus targets
```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Visit: http://localhost:9090/targets
```

### Check Grafana datasources
```bash
# Login to Grafana and go to Configuration → Data Sources
# Prometheus should be pre-configured
```

### View Prometheus logs
```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus -f
```

### View Grafana logs
```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -f
```

---

## Next Steps

1. Install Prometheus + Grafana using Helm
2. Access Grafana dashboard
3. Explore pre-built Kubernetes dashboards
4. Add Prometheus instrumentation to your FastAPI app
5. Create custom dashboards
6. Set up alerting rules
