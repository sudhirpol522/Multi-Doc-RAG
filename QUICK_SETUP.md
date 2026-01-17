# Quick Setup Commands for Minikube + ArgoCD

## 1. Start Minikube
```bash
minikube start --driver=docker --cpus=4 --memory=4096
```

## 2. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

## 3. Install ArgoCD Image Updater
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

## 4. Access ArgoCD UI
```bash
# Port-forward (run in separate terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password (Linux/Mac)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Get admin password (Windows PowerShell)
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
```

Access ArgoCD UI at: https://localhost:8080
- Username: admin
- Password: (from above command)

## 5. Create Secrets
```bash
kubectl create secret generic multi-doc-chat-secrets \
  --from-literal=google-api-key=YOUR_GOOGLE_API_KEY \
  --from-literal=groq-api-key=YOUR_GROQ_API_KEY \
  --from-literal=langsmith-api-key=YOUR_LANGSMITH_API_KEY \
  -n default
```

## 6. Apply ArgoCD Application
```bash
kubectl apply -f argocd/application.yaml
```

## 7. Access Your App
```bash
minikube service multi-doc-chat -n default
```

## GitHub Secrets Required
Add these in GitHub repo Settings → Secrets:
- DOCKERHUB_USERNAME: sudhirpol
- DOCKERHUB_TOKEN: (get from https://hub
.docker.com/settings/security)
