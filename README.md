# Version API - Canary Deployment Demo

A simple demo application to showcase Argo Rollouts canary deployment strategy on the IDP.

## Overview

This service returns a version string and demonstrates:
- **Canary Deployments** with Argo Rollouts
- **Gradual Traffic Shifting** (20% → 40% → 60% → 80% → 100%)
- **GitOps Workflow** with ArgoCD using Git tags
- **Multi-tenancy** on team-backend namespace

## Architecture

```
┌─────────────────┐
│  Argo Rollouts  │
│   Controller    │
└────────┬────────┘
         │
         ├──> Stable Pods (v1.0.0)
         └──> Canary Pods (v2.0.0) ──┐
                                      │
                            ┌─────────▼────────┐
                            │  Traffic Split   │
                            │  20% → 100%      │
                            └──────────────────┘
```

## Deployment Model

This app uses **tag-based deployments** with ArgoCD:

1. **Create a Git tag** for a new version (e.g., `v1.0.0`)
2. **ArgoCD detects the tag** and triggers deployment
3. **Argo Rollouts executes canary** strategy automatically
4. **Traffic shifts gradually**: 20% → 40% → 60% → 80% → 100%
5. **Full rollout** after all steps complete

### Why Tags?

- **Version Control**: Clear versioning with Git tags
- **Rollback**: Easy to revert to previous tag
- **Audit Trail**: Git history shows exact deployed versions
- **Best Practice**: Aligns with semantic versioning

## Quick Start

### 1. Check Initial Deployment

After ArgoCD syncs the app, verify the initial deployment:

```bash
# Check the rollout status
kubectl argo rollouts get rollout version-api -n team-backend

# Expected output:
# Name:            version-api
# Namespace:       team-backend
# Status:          ✔ Healthy
# Strategy:        Canary
# Images:          hashicorp/http-echo:0.2.3
# Replicas:
#   Desired:       4
#   Current:       4
#   Updated:       4
#   Ready:         4
#   Available:     4
```

### 2. Test the API

```bash
# Port forward to the service
kubectl port-forward -n team-backend svc/version-api 8080:80

# In another terminal, test the endpoint
curl localhost:8080
# Output: 🚀 Version 1.0.0 - Initial Release
```

### 3. Trigger a Canary Deployment

To demonstrate the canary rollout, update the version and create a Git tag:

```bash
# 1. Update the version in k8s/rollout.yaml
sed -i '' 's/Version 1.0.0/Version 2.0.0/g' k8s/rollout.yaml

# 2. Commit the change
git add k8s/rollout.yaml
git commit -m "feat: upgrade to v2.0.0"

# 3. Create a Git tag
git tag -a v2.0.0 -m "Release v2.0.0 - New Features"

# 4. Push the tag
git push origin v2.0.0
```

### 4. Watch the Canary Rollout

```bash
# Watch the rollout in real-time
kubectl argo rollouts get rollout version-api -n team-backend --watch

# You'll see:
# Step 1/4: 20% traffic to canary (1 pod) - wait 30s
# Step 2/4: 40% traffic to canary (2 pods) - wait 30s
# Step 3/4: 60% traffic to canary (3 pods) - wait 30s
# Step 4/4: 80% traffic to canary (4 pods) - wait 30s
# Completed: 100% traffic to v2.0.0
```

## Validation Methods

### 1. ArgoCD UI Validation

**URL**: https://argocd.devops.stxkxs.io/applications/backend-version-api

Check:
- ✅ **Sync Status**: Synced
- ✅ **Health Status**: Healthy
- ✅ **Git Revision**: Shows the deployed tag (e.g., `v2.0.0`)
- ✅ **Resources**: All resources (Rollout, Service, ConfigMap) healthy

### 2. Argo Rollouts CLI Validation

```bash
# Get rollout status
kubectl argo rollouts get rollout version-api -n team-backend

# Check revision history
kubectl argo rollouts history version-api -n team-backend

# Expected output:
# REVISION  TAG     CREATED
# 1         v1.0.0  2025-12-20 02:00:00
# 2         v2.0.0  2025-12-20 02:05:00 (current)
```

### 3. Kubectl Validation

```bash
# Check rollout resource
kubectl get rollout version-api -n team-backend

# Check pods
kubectl get pods -n team-backend -l app.kubernetes.io/name=version-api

# Expected: 4 pods running, all with v2.0.0 image

# Check services
kubectl get svc -n team-backend -l app.kubernetes.io/name=version-api

# Expected output:
# NAME                  TYPE        CLUSTER-IP     PORT(S)
# version-api           ClusterIP   10.0.x.x       80/TCP
# version-api-canary    ClusterIP   10.0.x.x       80/TCP

# Describe rollout to see events
kubectl describe rollout version-api -n team-backend

# Look for events like:
# - Updated Replicas: 4
# - Canary promoted to stable
# - Rollout completed successfully
```

### 4. Live Traffic Validation

During a canary rollout, test the traffic split:

```bash
# Port forward to the stable service
kubectl port-forward -n team-backend svc/version-api 8080:80 &

# Send multiple requests to see the mix
for i in {1..20}; do
  curl -s localhost:8080
  echo ""
done

# During canary at 20%:
# ~16 responses: Version 1.0.0 (stable)
# ~4 responses:  Version 2.0.0 (canary)

# During canary at 50%:
# ~10 responses: Version 1.0.0 (stable)
# ~10 responses: Version 2.0.0 (canary)

# After completion:
# All responses: Version 2.0.0
```

### 5. GitOps Validation

```bash
# Check the Application generated by ApplicationSet
kubectl get application backend-version-api -n argocd -o yaml

# Verify:
# - spec.source.targetRevision points to the Git tag
# - spec.source.repoURL points to this repo
# - spec.destination.namespace is team-backend
# - status.sync.status is "Synced"
# - status.health.status is "Healthy"
```

### 6. ApplicationSet Validation

```bash
# Check the ApplicationSet that generates this app
kubectl get applicationset team-apps -n argocd -o yaml

# Verify it picks up the app from teams/backend/team.yaml:
# - generators.git.files.path: "teams/*/team.yaml"
# - template references the correct repo and tag
```

### 7. Resource Quota Validation

```bash
# Verify the app respects team resource quotas
kubectl get resourcequota -n team-backend

# Check current usage
kubectl describe resourcequota backend-quota -n team-backend

# Expected (for 4 version-api pods):
# requests.cpu:     200m/20 (4 pods × 50m each)
# requests.memory:  256Mi/40Gi (4 pods × 64Mi each)
# pods:             4/50
```

### 8. Network Policy Validation

```bash
# Check that network policies allow traffic
kubectl get networkpolicy -n team-backend

# Test connectivity from another namespace (should work from allowed namespaces)
kubectl run -it --rm debug -n monitoring --image=curlimages/curl --restart=Never \
  -- curl http://version-api.team-backend/

# Output: Version X.X.X (successful)

# Test from disallowed namespace (should timeout)
kubectl run -it --rm debug -n default --image=curlimages/curl --restart=Never \
  -- curl --max-time 5 http://version-api.team-backend/

# Output: timeout (blocked by network policy)
```

## Release Process

### Creating a New Release

```bash
# 1. Update version in rollout.yaml
vim k8s/rollout.yaml
# Change: "-text=🚀 Version X.X.X ..."

# 2. Commit changes
git add k8s/rollout.yaml
git commit -m "feat: release vX.X.X"

# 3. Create annotated tag
git tag -a vX.X.X -m "Release vX.X.X - Description"

# 4. Push commit and tag
git push origin main
git push origin vX.X.X

# 5. ArgoCD detects the tag and deploys
# Watch: https://argocd.devops.stxkxs.io
```

## Links

- [ArgoCD Dashboard](https://argocd.devops.stxkxs.io)
- [Argo Rollouts Dashboard](https://rollouts.devops.stxkxs.io)
- [Backstage Catalog](https://backstage.devops.stxkxs.io)
