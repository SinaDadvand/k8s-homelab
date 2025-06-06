# ArgoCD Application Examples

This directory contains three different examples of ArgoCD Application configurations, demonstrating various source repository scenarios for GitOps deployments.

## Files Overview

### 1. `17-argocd-application.yaml` - Local Repository Example
- **Purpose**: Demonstrates using your own Git repository with local applications
- **Source**: `https://github.com/sinadogru/homelab.git`
- **Path**: `gitops-apps/nginx-app`
- **Application**: Simple nginx deployment
- **Namespace**: `nginx-demo`

### 2. `17b-argocd-application-public-repo.yaml` - Public Repository Example
- **Purpose**: Demonstrates using a well-known public repository
- **Source**: `https://github.com/argoproj/argocd-example-apps.git`
- **Path**: `guestbook`
- **Application**: ArgoCD's classic guestbook demo
- **Namespace**: `guestbook`

### 3. `17c-argocd-application-personal-repo.yaml` - Personal Repository Examples
- **Purpose**: Shows how to use your personal Git repository for multiple applications
- **Contains two applications**:
  1. Redis demo (`redis-app`)
  2. Multi-tier application demo (`multi-tier-app`)
- **Source**: `https://github.com/sinadogru/homelab.git`
- **Namespaces**: `redis-demo` and `multitier-demo`

## Usage Instructions

### Prerequisites
1. ArgoCD installed and running in your cluster
2. ArgoCD CLI configured (optional but recommended)
3. kubectl access to your cluster

### Deploying Applications

#### Option 1: Using kubectl
```bash
# Deploy the local nginx app
kubectl apply -f 17-argocd-application.yaml

# Deploy the public repo guestbook app
kubectl apply -f 17b-argocd-application-public-repo.yaml

# Deploy personal repo applications
kubectl apply -f 17c-argocd-application-personal-repo.yaml
```

#### Option 2: Using ArgoCD CLI
```bash
# Create applications using ArgoCD CLI
argocd app create -f 17-argocd-application.yaml
argocd app create -f 17b-argocd-application-public-repo.yaml
argocd app create -f 17c-argocd-application-personal-repo.yaml

# Sync applications
argocd app sync local-nginx-demo
argocd app sync public-repo-demo
argocd app sync personal-repo-demo
argocd app sync personal-multitier-demo
```

### Viewing Applications
1. **ArgoCD UI**: Access the ArgoCD web interface to view application status
2. **CLI**: Use `argocd app list` and `argocd app get <app-name>`
3. **kubectl**: Check application resources with `kubectl get applications -n argocd`

## Key Configuration Differences

### Sync Policy
All examples use automated sync with:
- **prune: true** - Removes resources not defined in Git
- **selfHeal: true** - Automatically corrects configuration drift
- **CreateNamespace=true** - Creates target namespace if missing

### Repository Types
1. **Local/Personal Repo**: Uses your own repository for full control
2. **Public Repo**: Uses community examples for learning
3. **Mixed Approach**: Combines both strategies for different applications

## Troubleshooting

### Common Issues
1. **Repository Access**: Ensure the Git repository is publicly accessible or configure proper credentials
2. **Path Errors**: Verify the `path` field points to a directory containing valid Kubernetes manifests
3. **Namespace Conflicts**: Check if target namespaces already exist with conflicting resources

### Validation Commands
```bash
# Check application status
kubectl get applications -n argocd

# View application details
kubectl describe application <app-name> -n argocd

# Check ArgoCD server logs
kubectl logs -n argocd deployment/argocd-server
```

## Next Steps
1. Customize the repository URLs to point to your own Git repositories
2. Modify application paths to match your repository structure
3. Adjust sync policies based on your deployment requirements
4. Add health checks and custom sync hooks as needed
