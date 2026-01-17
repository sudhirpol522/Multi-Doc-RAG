# Complete Monitoring Setup Guide - Step by Step

## Step 1: Install Helm

### Windows (PowerShell as Administrator)

#### Option A: Using Chocolatey
```powershell
choco install kubernetes-helm
```

#### Option B: Using Scoop
```powershell
scoop install helm
```

#### Option C: Manual Installation
1. Download from: https://github.com/helm/helm/releases
2. Extract the zip file
3. Add helm.exe to your PATH

### Verify Installation
```bash
helm version
```

---

## Step 2: Install Prometheus + Grafana Stack

```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the complete monitoring stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123 \
  --timeout 10m

# This installs:
# - Prometheus (metrics collection)
# - Grafana (dashboards and visualization)
# - Alertmanager (alerting)
# - Node Exporter (node metrics)
# - Kube State Metrics (k8s resource metrics)
```

---

## Step 3: Wait for Pods to be Ready

```bash
# Check installation status
kubectl get pods -n monitoring

# Wait for all pods to be running (takes 2-3 minutes)
kubectl wait --for=condition=Ready pods --all -n monitoring --timeout=300s
```

You should see output like:
```
NAME                                                   READY   STATUS
alertmanager-monitoring-kube-prometheus-alertmanager   2/2     Running
monitoring-grafana-xxxxx                               3/3     Running
monitoring-kube-prometheus-operator-xxxxx              1/1     Running
monitoring-kube-state-metrics-xxxxx                    1/1     Running
monitoring-prometheus-node-exporter-xxxxx              1/1     Running
prometheus-monitoring-kube-prometheus-prometheus       2/2     Running
```

---

## Step 4: Access Grafana Dashboard

### Port-Forward (run in a new terminal)
```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

### Access
- **URL**: http://localhost:3000
- **Username**: admin
- **Password**: admin123

---

## Step 5: Access Prometheus UI

### Port-Forward (run in another terminal)
```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

### Access
- **URL**: http://localhost:9090

---

## Step 6: Explore Pre-built Dashboards

In Grafana (http://localhost:3000), navigate to **Dashboards** and explore:

1. **Kubernetes / Compute Resources / Cluster**
   - Overall cluster CPU, Memory, Network usage
   
2. **Kubernetes / Compute Resources / Namespace (Pods)**
   - Resource usage per namespace
   
3. **Kubernetes / Compute Resources / Pod**
   - Individual pod metrics
   
4. **Node Exporter / Nodes**
   - Node-level system metrics

---

## Step 7: Monitor Your MultiDocChat Application

### View Your App Metrics

1. In Grafana, go to **Explore**
2. Select **Prometheus** as data source
3. Try these queries:

```promql
# CPU usage
rate(container_cpu_usage_seconds_total{pod=~"multi-doc-chat.*"}[5m])

# Memory usage (MB)
container_memory_usage_bytes{pod=~"multi-doc-chat.*"} / 1024 / 1024

# Network received (bytes/sec)
rate(container_network_receive_bytes_total{pod=~"multi-doc-chat.*"}[5m])

# Network transmitted (bytes/sec)
rate(container_network_transmit_bytes_total{pod=~"multi-doc-chat.*"}[5m])

# Pod restart count
kube_pod_container_status_restarts_total{pod=~"multi-doc-chat.*"}
```

---

## Step 8: Create Custom Dashboard (Optional)

1. In Grafana, click **+** → **Import Dashboard**
2. Upload the dashboard JSON: `monitoring/grafana-dashboard-multi-doc-chat.json`
3. Select **Prometheus** as data source
4. Click **Import**

---

## Step 9: Add Prometheus Metrics to Your App (Optional)

To expose custom application metrics:

### Update main.py
```python
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Response

# Define metrics
http_requests_total = Counter(
    'http_requests_total', 
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint']
)

@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.middleware("http")
async def add_metrics(request: Request, call_next):
    """Middleware to track request metrics"""
    start_time = time.time()
    
    response = await call_next(request)
    
    # Track metrics
    duration = time.time() - start_time
    http_requests_total.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    http_request_duration_seconds.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    
    return response
```

### Update requirements.txt
```
prometheus-client==0.20.0
```

### Rebuild and push image
```bash
docker build -t sudhirpol/multi-doc-chat:latest .
docker push sudhirpol/multi-doc-chat:latest

# Restart pod
kubectl delete pod -l app=multi-doc-chat -n default
```

---

## Useful Commands

### Check Prometheus Targets
```bash
# Access Prometheus UI
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Go to: http://localhost:9090/targets
# You should see all monitored endpoints
```

### View Logs
```bash
# Prometheus logs
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus -f

# Grafana logs
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -f

# Alertmanager logs
kubectl logs -n monitoring -l app.kubernetes.io/name=alertmanager -f
```

### Get Service URLs
```bash
# All monitoring services
kubectl get svc -n monitoring
```

---

## Troubleshooting

### Pods not starting
```bash
# Check pod status
kubectl get pods -n monitoring

# Describe problematic pod
kubectl describe pod <pod-name> -n monitoring

# Check events
kubectl get events -n monitoring --sort-by='.lastTimestamp'
```

### Grafana not accessible
```bash
# Check Grafana service
kubectl get svc monitoring-grafana -n monitoring

# Check Grafana pod
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana
```

### Prometheus not scraping metrics
```bash
# Check ServiceMonitor
kubectl get servicemonitor -n monitoring

# Check Prometheus config
kubectl get prometheus -n monitoring -o yaml
```

---

## Cleanup

```bash
# Uninstall monitoring stack
helm uninstall monitoring -n monitoring

# Delete namespace
kubectl delete namespace monitoring
```

---

## What You Get

✅ **Prometheus** - Metrics collection and storage
✅ **Grafana** - Beautiful dashboards and visualization
✅ **Alertmanager** - Alert routing and management
✅ **Pre-built Dashboards** - 15+ Kubernetes dashboards
✅ **Node Metrics** - CPU, memory, disk, network per node
✅ **Pod Metrics** - Resource usage per pod
✅ **Cluster Metrics** - Overall cluster health
✅ **Custom Metrics** - Add your own application metrics

---

## Next Steps

1. Install Helm (if not installed)
2. Run the installation command
3. Access Grafana at http://localhost:3000
4. Explore pre-built dashboards
5. Add custom metrics to your FastAPI app
6. Create custom dashboards
7. Set up alerts

Happy Monitoring! 📊🎉
