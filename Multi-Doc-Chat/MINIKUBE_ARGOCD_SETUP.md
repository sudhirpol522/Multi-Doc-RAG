# Local Kubernetes Setup with Minikube + ArgoCD + Image Updater

## Architecture Overview

```
GitHub Push → CI (Build & Push to Docker Hub) → ArgoCD Image Updater → Updates Manifests → ArgoCD Syncs → Minikube
```

## Prerequisites

- Docker Desktop installed
- Minikube installed
- kubectl installed
- ArgoCD CLI installed

---

## Step 1: Install Minikube

### Windows (PowerShell)
```powershell
# Download minikube
New-Item -Path 'c:\' -Name 'minikube' -ItemType Directory -Force
Invoke-WebRequest -OutFile 'c:\minikube\minikube.exe' -Uri 'https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe' -UseBasicParsing

# Add to PATH
$oldPath = [Environment]::GetEnvironmentVariable('Path', [EnvironmentVariableTarget]::Machine)
if ($oldPath.Split(';') -inotcontains 'C:\minikube'){
  [Environment]::SetEnvironmentVariable('Path', $('{0};C:\minikube' -f $oldPath), [EnvironmentVariableTarget]::Machine)
}
```

### Start Minikube
```bash
# Start minikube with Docker driver
minikube start --driver=docker --cpus=4 --memory=4096

# Verify
kubectl cluster-info
kubectl get nodes
```

---

## Step 2: Install ArgoCD on Minikube

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Port-forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# In a new terminal, get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# Or on Windows PowerShell:
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))

# Access ArgoCD UI at: https://localhost:8080
# Username: admin
# Password: (from above command)
```

---

## Step 3: Install ArgoCD Image Updater

```bash
# Install Image Updater
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Verify installation
kubectl get pods -n argocd | grep image-updater

# Check logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f
```

---

## Step 4: Create Kubernetes Manifests

Create directory structure:
```
Multi-Doc-Chat/
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── secret.yaml
```

### Create k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-doc-chat
  labels:
    app: multi-doc-chat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multi-doc-chat
  template:
    metadata:
      labels:
        app: multi-doc-chat
    spec:
      containers:
      - name: multi-doc-chat
        image: sudhirpol/multi-doc-chat:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: GOOGLE_API_KEY
          valueFrom:
            secretKeyRef:
              name: multi-doc-chat-secrets
              key: google-api-key
        - name: GROQ_API_KEY
          valueFrom:
            secretKeyRef:
              name: multi-doc-chat-secrets
              key: groq-api-key
        - name: LANGSMITH_API_KEY
          valueFrom:
            secretKeyRef:
              name: multi-doc-chat-secrets
              key: langsmith-api-key
        - name: LLM_PROVIDER
          valueFrom:
            configMapKeyRef:
              name: multi-doc-chat-config
              key: llm-provider
        - name: ENV
          value: "local"
        - name: PORT
          value: "8080"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Create k8s/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-doc-chat
  labels:
    app: multi-doc-chat
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
    protocol: TCP
    name: http
  selector:
    app: multi-doc-chat
```

### Create k8s/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: multi-doc-chat-config
data:
  llm-provider: "google"
  langsmith-endpoint: "https://api.smith.langchain.com"
  langsmith-tracing: "true"
```

### Create k8s/secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: multi-doc-chat-secrets
type: Opaque
stringData:
  google-api-key: "YOUR_GOOGLE_API_KEY"
  groq-api-key: "YOUR_GROQ_API_KEY"
  langsmith-api-key: "YOUR_LANGSMITH_API_KEY"
```

**Important:** Don't commit this file with real secrets! Use Sealed Secrets or don't commit it.

---

## Step 5: Update CI Workflow to Push to Docker Hub

Update `.github/workflows/ci.yml`:

```yaml
name: CI - Build and Push to Docker Hub

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  DOCKER_IMAGE: sudhirpol/multi-doc-chat

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Set up uv
        uses: astral-sh/setup-uv@v4

      - name: Sync dependencies with uv (locked)
        run: |
          uv sync --frozen
          uv pip install pytest

      - name: Run tests
        env:
          PYTHONPATH: ${{ github.workspace }}/multi_doc_chat
          GROQ_API_KEY: dummy
          GOOGLE_API_KEY: dummy
          LLM_PROVIDER: google
        run: |
          uv run pytest -q

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.DOCKER_IMAGE }}
          tags: |
            type=sha,prefix={{branch}}-
            type=raw,value=latest

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache,mode=max

      - name: Image digest
        run: echo ${{ steps.meta.outputs.digest }}
```

---

## Step 6: Configure Docker Hub Secrets in GitHub

1. Go to your GitHub repository
2. Settings → Secrets and variables → Actions → New repository secret
3. Add these secrets:
   - `DOCKERHUB_USERNAME`: Your Docker Hub username (sudhirpol)
   - `DOCKERHUB_TOKEN`: Your Docker Hub access token

**To create Docker Hub token:**
1. Go to https://hub.docker.com/settings/security
2. Click "New Access Token"
3. Give it a name (e.g., "github-actions")
4. Copy the token and add to GitHub secrets

---

## Step 7: Create ArgoCD Application with Image Updater

Create `argocd/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-doc-chat
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: multi-doc-chat=sudhirpol/multi-doc-chat:latest
    argocd-image-updater.argoproj.io/multi-doc-chat.update-strategy: latest
    argocd-image-updater.argoproj.io/multi-doc-chat.allow-tags: regexp:^main-.*$
    argocd-image-updater.argoproj.io/write-back-method: git
spec:
  project: default
  source:
    repoURL: https://github.com/sudhirpol522/Multi-Doc-RAG.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Apply the application:
```bash
kubectl apply -f argocd/application.yaml
```

---

## Step 8: Create Secrets in Minikube

```bash
# Create namespace (if needed)
kubectl create namespace default

# Create secrets
kubectl create secret generic multi-doc-chat-secrets \
  --from-literal=google-api-key=YOUR_GOOGLE_API_KEY \
  --from-literal=groq-api-key=YOUR_GROQ_API_KEY \
  --from-literal=langsmith-api-key=YOUR_LANGSMITH_API_KEY \
  -n default

# Verify
kubectl get secrets -n default
```

---

## Step 9: Access Your Application

### Method 1: Using Minikube Service
```bash
# This will open the service in your browser
minikube service multi-doc-chat -n default
```

### Method 2: Using kubectl port-forward
```bash
kubectl port-forward svc/multi-doc-chat 8000:80 -n default
# Access at http://localhost:8000
```

### Method 3: Get Minikube IP and NodePort
```bash
# Get minikube IP
minikube ip

# Access at http://<MINIKUBE_IP>:30080
```

---

## Step 10: Verify Everything Works

### Check ArgoCD Application
```bash
# Using ArgoCD UI at https://localhost:8080
# Or using CLI:
argocd app get multi-doc-chat
argocd app sync multi-doc-chat
```

### Check Pods
```bash
kubectl get pods -n default
kubectl logs -l app=multi-doc-chat -f
```

### Check Image Updater
```bash
# Check if image updater is running
kubectl get pods -n argocd | grep image-updater

# Check logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f
```

### Test Health Endpoint
```bash
# Get the service URL
minikube service multi-doc-chat -n default --url

# Test
curl $(minikube service multi-doc-chat -n default --url)/health
```

---

## Complete Workflow

1. **Make code changes** and push to GitHub
2. **CI runs automatically:**
   - Runs tests
   - Builds Docker image
   - Pushes to Docker Hub with tag `main-<commit-sha>` and `latest`
3. **ArgoCD Image Updater detects new image** (checks every 2 minutes by default)
4. **Updates the deployment.yaml** in your repo with the new image tag
5. **ArgoCD detects the change** and syncs to Minikube
6. **New pod rolls out** automatically

---

## Troubleshooting

### Check ArgoCD Image Updater logs
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f
```

### Manually trigger ArgoCD sync
```bash
argocd app sync multi-doc-chat
```

### Check if image exists in Docker Hub
```bash
docker pull sudhirpol/multi-doc-chat:latest
```

### Restart ArgoCD Image Updater
```bash
kubectl rollout restart deployment argocd-image-updater -n argocd
```

### Check pod events
```bash
kubectl describe pod -l app=multi-doc-chat -n default
```

---

## Clean Up

### Stop Minikube
```bash
minikube stop
```

### Delete Minikube cluster
```bash
minikube delete
```

### Delete ArgoCD application
```bash
kubectl delete -f argocd/application.yaml
```

---

## Next Steps

1. Create Docker Hub account/token
2. Add secrets to GitHub
3. Create k8s manifests
4. Start Minikube
5. Install ArgoCD and Image Updater
6. Apply ArgoCD application
7. Push code changes and watch auto-deployment!

This setup gives you a complete local GitOps workflow! 🚀
