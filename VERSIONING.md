# Semantic Versioning & Automated Deployment

## Overview

This project uses **Semantic Versioning (SemVer)** for Docker images with automated version bumping via GitHub Actions CI/CD pipeline. ArgoCD Image Updater monitors Docker Hub for new versions matching the `1.*` pattern and automatically deploys them to Kubernetes.

## Versioning Strategy

**Format**: `MAJOR.MINOR.PATCH` (e.g., `1.0.0`, `1.0.1`, `1.1.0`)

- **MAJOR**: Breaking changes (currently `1`)
- **MINOR**: New features, backwards-compatible
- **PATCH**: Bug fixes, backwards-compatible

## How It Works

### 1. CI Pipeline Auto-Versioning

When you push to `main` branch:

1. **Tests run** using pytest with dummy API keys
2. **Version is auto-incremented**:
   - Fetches latest tag matching `1.*` from Git
   - If no tags exist, starts with `1.0.0`
   - Increments PATCH version (e.g., `1.0.5` → `1.0.6`)
3. **Docker image is built and pushed** with two tags:
   - `sudhirpol/multi-doc-chat:1.0.6` (versioned)
   - `sudhirpol/multi-doc-chat:latest` (convenience)
4. **Git tag is created** and pushed to repository

### 2. ArgoCD Image Updater Deployment

ArgoCD Image Updater is configured to:

- **Watch** Docker Hub for images matching `sudhirpol/multi-doc-chat:1.*`
- **Use semver strategy** to find the highest version
- **Automatically update** `k8s/deployment.yaml` with new image tag
- **Commit changes** back to Git repository
- **Trigger ArgoCD sync** to deploy to Minikube

## Configuration Files

### `.github/workflows/ci.yml`

```yaml
- name: Get latest tag
  id: get_version
  run: |
    LATEST_TAG=$(git tag -l "1.*" --sort=-v:refname | head -n 1)
    if [ -z "$LATEST_TAG" ]; then
      NEW_VERSION="1.0.0"
    else
      MAJOR=$(echo $LATEST_TAG | cut -d. -f1)
      MINOR=$(echo $LATEST_TAG | cut -d. -f2)
      PATCH=$(echo $LATEST_TAG | cut -d. -f3)
      NEW_PATCH=$((PATCH + 1))
      NEW_VERSION="${MAJOR}.${MINOR}.${NEW_PATCH}"
    fi
    echo "version=$NEW_VERSION" >> $GITHUB_OUTPUT
```

### `argocd/application.yaml`

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: multi-doc-chat=sudhirpol/multi-doc-chat
  argocd-image-updater.argoproj.io/multi-doc-chat.update-strategy: semver
  argocd-image-updater.argoproj.io/multi-doc-chat.allow-tags: regexp:^1\..*$
```

### `k8s/deployment.yaml`

```yaml
containers:
- name: multi-doc-chat
  image: sudhirpol/multi-doc-chat:1.0.0  # Updated by ArgoCD Image Updater
```

## Workflow Example

```
┌─────────────────────────────────────────────────────────────────┐
│ Developer pushes code to main                                   │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ GitHub Actions CI                                               │
│ 1. Runs pytest tests                                            │
│ 2. Gets latest tag (e.g., 1.0.5)                               │
│ 3. Increments to 1.0.6                                          │
│ 4. Builds Docker image                                          │
│ 5. Pushes to Docker Hub with tags: 1.0.6, latest              │
│ 6. Creates Git tag 1.0.6                                        │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ ArgoCD Image Updater (polls Docker Hub every 2 min)            │
│ 1. Detects new image 1.0.6                                      │
│ 2. Updates k8s/deployment.yaml                                  │
│ 3. Commits change to Git                                        │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ ArgoCD                                                           │
│ 1. Detects Git manifest change                                  │
│ 2. Syncs to Minikube cluster                                    │
│ 3. Rolling update with new image                                │
└─────────────────────────────────────────────────────────────────┘
```

## Manual Version Control

### To Manually Set Version

If you need to skip versions or set a specific version:

```bash
# Create and push a specific tag
git tag 1.1.0
git push origin 1.1.0

# Next auto-increment will be 1.1.1
```

### To Bump MINOR Version

```bash
# Manually create a new minor version
git tag 1.1.0
git push origin 1.1.0

# CI will auto-increment to 1.1.1, 1.1.2, etc.
```

### To Check Current Version

```bash
# Check deployed version in Kubernetes
kubectl get deployment multi-doc-chat -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check latest Git tag
git tag -l "1.*" --sort=-v:refname | head -n 1

# Check available versions on Docker Hub
docker search sudhirpol/multi-doc-chat
curl -s https://hub.docker.com/v2/repositories/sudhirpol/multi-doc-chat/tags | jq -r '.results[].name'
```

## Rollback Strategy

### Rollback to Previous Version

```bash
# View deployment history
kubectl rollout history deployment/multi-doc-chat

# Rollback to previous version
kubectl rollout undo deployment/multi-doc-chat

# Rollback to specific version
kubectl rollout undo deployment/multi-doc-chat --to-revision=2
```

### Manual Rollback by Changing Image

```bash
# Update deployment to specific version
kubectl set image deployment/multi-doc-chat multi-doc-chat=sudhirpol/multi-doc-chat:1.0.5

# Or edit the k8s/deployment.yaml and commit
git checkout k8s/deployment.yaml
# Edit image tag to desired version
git commit -m "Rollback to version 1.0.5"
git push
```

## Monitoring Deployments

### Check ArgoCD Image Updater Status

```bash
# Check image updater logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f

# Check ArgoCD application status
argocd app get multi-doc-chat
```

### Check Deployment Status

```bash
# Check rollout status
kubectl rollout status deployment/multi-doc-chat

# Check pod image version
kubectl get pods -l app=multi-doc-chat -o jsonpath='{.items[*].spec.containers[*].image}'

# Check recent deployment events
kubectl describe deployment multi-doc-chat
```

## GitHub Secrets Required

Ensure these secrets are configured in GitHub repository settings:

- `DOCKERHUB_USERNAME`: Your Docker Hub username (e.g., `sudhirpol`)
- `DOCKERHUB_TOKEN`: Docker Hub access token

**To create Docker Hub token:**
1. Go to https://hub.docker.com/settings/security
2. Click "New Access Token"
3. Name it (e.g., "GitHub Actions")
4. Copy the token and add to GitHub secrets

## Troubleshooting

### CI Pipeline Fails on Tag Creation

**Error**: `fatal: tag '1.0.6' already exists`

**Solution**: Delete the tag and re-run CI
```bash
git tag -d 1.0.6
git push origin --delete 1.0.6
```

### ArgoCD Not Detecting New Image

**Check Image Updater logs:**
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater --tail=50
```

**Common issues:**
- Image Updater not running: `kubectl get pods -n argocd`
- Wrong image name in annotations
- Git write-back credentials missing

### Deployment Stuck in Pending

**Check pod status:**
```bash
kubectl get pods -l app=multi-doc-chat
kubectl describe pod <pod-name>
```

**Common issues:**
- ImagePullBackOff: Wrong image tag or registry credentials
- Resource constraints: Insufficient CPU/memory
- Secrets missing: Create `multi-doc-chat-secrets`

## Best Practices

1. **Always test locally** before pushing to main
2. **Use feature branches** for development
3. **Monitor CI pipeline** after pushing to main
4. **Watch ArgoCD** for automatic deployment
5. **Check pod logs** after deployment: `kubectl logs -l app=multi-doc-chat`
6. **Use semantic versioning properly**:
   - PATCH: Bug fixes (1.0.x)
   - MINOR: New features (1.x.0)
   - MAJOR: Breaking changes (x.0.0)

## Future Enhancements

- Add manual approval step for production deployments
- Implement blue-green or canary deployment strategies
- Add automated smoke tests after deployment
- Integrate with Slack/Discord for deployment notifications
- Add promotion workflow for staging → production
