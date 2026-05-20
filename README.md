# Quick Reference Commands

## 🚀 Quick Setup Commands

### 1. Generate GitHub Personal Access Token (PAT)
```
Go to: https://github.com/settings/tokens
1. Click "Generate new token (classic)"
2. Name: GITOPS_PAT
3. Select: repo, workflow
4. Generate token and save it
```

### 2. Add Secret to GitHub Repository
```
Go to: https://github.com/ilyasjaelani/simple-loginapp/settings/secrets/actions
1. Click "New repository secret"
2. Name: GITOPS_PAT
3. Value: <paste your token>
```

### 3. One-time Kubernetes Setup
```bash
# Create namespace
kubectl create namespace simple-loginapp

# Apply database secret (update the values first!)
kubectl apply -f apps/simple-loginapp/db-secret.yaml

# Deploy ArgoCD application
kubectl apply -f argocd/application.yaml
```

## 📦 File Placement

### In `simple-loginapp` repository:
```bash
# Add workflow
.github/workflows/ci-cd-pipeline.yaml

# Add Dockerfiles
frontend/Dockerfile
backend/Dockerfile
```

### In `simple-loginapp-gitops` repository:
```bash
# Add K8s manifests
apps/simple-loginapp/namespace.yaml
apps/simple-loginapp/frontend-deployment.yaml
apps/simple-loginapp/backend-deployment.yaml
apps/simple-loginapp/db-secret.yaml

# Add ArgoCD config
argocd/application.yaml
```

## 🔍 Monitoring Commands

### Check GitHub Actions
```bash
# Via web: https://github.com/ilyasjaelani/simple-loginapp/actions
```

### Check ArgoCD
```bash
# Get ArgoCD password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443

# CLI - Get app status
kubectl get application simple-loginapp -n argocd

# CLI - Sync manually (if needed)
kubectl patch application simple-loginapp -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"syncStrategy":{"hook":{}}}}}'
```

### Check Kubernetes Resources
```bash
# All resources
kubectl get all -n simple-loginapp

# Pods
kubectl get pods -n simple-loginapp -w

# Deployments
kubectl get deployments -n simple-loginapp

# Services
kubectl get svc -n simple-loginapp

# Pod logs
kubectl logs -f deployment/frontend -n simple-loginapp
kubectl logs -f deployment/backend -n simple-loginapp

# Describe resources (troubleshooting)
kubectl describe pod <pod-name> -n simple-loginapp
kubectl describe deployment frontend -n simple-loginapp

# Events
kubectl get events -n simple-loginapp --sort-by='.lastTimestamp'
```

### Check Container Registry
```bash
# View packages: https://github.com/ilyasjaelani?tab=packages

# Pull image manually (testing)
docker pull ghcr.io/ilyasjaelani/simple-loginapp-frontend:latest
docker pull ghcr.io/ilyasjaelani/simple-loginapp-backend:latest
```

## 🧪 Testing the Pipeline

### Trigger CI/CD
```bash
# Make a change in simple-loginapp repo
cd simple-loginapp
echo "// pipeline test" >> frontend/server.js
git add .
git commit -m "test: trigger pipeline"
git push origin main
```

### Watch the process
```bash
# Terminal 1: Watch GitHub Actions
# Go to: https://github.com/ilyasjaelani/simple-loginapp/actions

# Terminal 2: Watch ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080

# Terminal 3: Watch Pods
kubectl get pods -n simple-loginapp -w

# Terminal 4: Watch Events
watch -n 2 'kubectl get events -n simple-loginapp --sort-by=".lastTimestamp" | tail -20'
```

## 🔄 Manual Operations

### Force ArgoCD Sync
```bash
# Via CLI
kubectl patch application simple-loginapp -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'

# Or via UI: Click "Sync" button in ArgoCD dashboard
```

### Rollback Deployment
```bash
# Kubernetes rollback
kubectl rollout undo deployment/frontend -n simple-loginapp
kubectl rollout undo deployment/backend -n simple-loginapp

# Check rollout status
kubectl rollout status deployment/frontend -n simple-loginapp
```

### Scale Deployment
```bash
# Scale up/down
kubectl scale deployment frontend --replicas=3 -n simple-loginapp
kubectl scale deployment backend --replicas=3 -n simple-loginapp

# Note: ArgoCD will reset this if selfHeal is enabled
```

### Update Secret
```bash
# Edit secret
kubectl edit secret db-credentials -n simple-loginapp

# Or recreate
kubectl delete secret db-credentials -n simple-loginapp
kubectl apply -f apps/simple-loginapp/db-secret.yaml
```

## 🐛 Debugging

### Pod not starting
```bash
# Check pod status
kubectl get pods -n simple-loginapp

# Describe the pod
kubectl describe pod <pod-name> -n simple-loginapp

# Check logs
kubectl logs <pod-name> -n simple-loginapp

# Check previous container logs (if crashed)
kubectl logs <pod-name> -n simple-loginapp --previous
```

### Image pull error
```bash
# Check if image exists
docker pull ghcr.io/ilyasjaelani/simple-loginapp-frontend:latest

# Check image pull secret (if using private registry)
kubectl get secrets -n simple-loginapp

# Describe pod to see exact error
kubectl describe pod <pod-name> -n simple-loginapp
```

### ArgoCD not syncing
```bash
# Check application status
kubectl describe application simple-loginapp -n argocd

# View application in UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Force refresh
kubectl delete application simple-loginapp -n argocd
kubectl apply -f argocd/application.yaml
```

### Database connection issues
```bash
# Check if secret exists
kubectl get secret db-credentials -n simple-loginapp -o yaml

# Check backend logs
kubectl logs -f deployment/backend -n simple-loginapp

# Test database connection from pod
kubectl exec -it deployment/backend -n simple-loginapp -- sh
# Inside pod:
# env | grep DB
# ping $DB_HOST
```

## 🔐 Security Commands

### Create PAT with minimal permissions
```
Scopes needed:
- repo (full control of private repositories)
- workflow (update GitHub Action workflows)
```

### Rotate secrets
```bash
# Delete old secret
kubectl delete secret db-credentials -n simple-loginapp

# Create new secret
kubectl create secret generic db-credentials \
  -n simple-loginapp \
  --from-literal=host='new-host' \
  --from-literal=username='new-user' \
  --from-literal=password='new-password' \
  --from-literal=database='new-db'

# Restart deployments to use new secret
kubectl rollout restart deployment/backend -n simple-loginapp
```

## 📊 Useful Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc

# Quick namespace switch
alias kn='kubectl config set-context --current --namespace'

# Simple loginapp namespace
alias ksl='kubectl -n simple-loginapp'

# Get all
alias kga='kubectl get all -n simple-loginapp'

# Logs
alias klf='kubectl logs -f -n simple-loginapp'

# ArgoCD port forward
alias argocd-ui='kubectl port-forward svc/argocd-server -n argocd 8080:443'
```

## 🎯 Common Workflows

### Deploy new version
```bash
# 1. Make code changes in simple-loginapp
# 2. Commit and push to main
git add .
git commit -m "feat: add new feature"
git push origin main

# 3. GitHub Actions will automatically:
#    - Build new images
#    - Push to ghcr.io
#    - Update gitops repo

# 4. ArgoCD will automatically:
#    - Detect changes
#    - Sync to cluster
#    - Deploy new version

# 5. Monitor deployment
kubectl get pods -n simple-loginapp -w
```

### Emergency rollback
```bash
# Option 1: Kubernetes rollback
kubectl rollout undo deployment/frontend -n simple-loginapp
kubectl rollout undo deployment/backend -n simple-loginapp

# Option 2: Git revert in gitops repo
cd simple-loginapp-gitops
git log --oneline
git revert <commit-hash>
git push origin main
# ArgoCD will auto-sync the revert
```

### Check deployment health
```bash
# Overall status
kubectl get all -n simple-loginapp

# Detailed deployment status
kubectl describe deployment frontend -n simple-loginapp
kubectl describe deployment backend -n simple-loginapp

# Pod health
kubectl get pods -n simple-loginapp
kubectl top pods -n simple-loginapp

# Service endpoints
kubectl get endpoints -n simple-loginapp

# Recent events
kubectl get events -n simple-loginapp --sort-by='.lastTimestamp' | tail -20
```

---
**Quick Start**: Copy files → Set PAT → Push to GitHub → Watch it deploy! 🚀