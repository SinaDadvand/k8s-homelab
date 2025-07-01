# GitOps Advanced Learning Path - Troubleshooting Guide
**Common Issues and Solutions for Phase 1 & Phase 2**

## 📋 Overview
This document tracks common issues encountered during the GitOps Advanced Learning Path (both Phase 1 and Phase 2) along with their solutions and troubleshooting steps.

---

## 🔧 Phase 1 Issues (Steps 20-26)

### Issue #1: Missing Helm Template Values
**Step:** 23-24 (Multi-Environment Helm Deployment)  
**Symptoms:**
- ArgoCD sync fails with template errors
- Error messages like: `nil pointer evaluating interface {}.create`
- Templates for `serviceaccount.yaml` or `hpa.yaml` fail to render

**Root Cause:** Custom `values.yaml` missing required sections for auto-generated Helm templates

**Solution:**
```yaml
# Add these sections to helm-charts/multi-env-app/values.yaml
serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
```

**Troubleshooting Commands:**
```powershell
# Test Helm template rendering locally
helm template staging helm-charts/multi-env-app -f helm-charts/multi-env-app/values-staging.yaml --debug

# Check for missing values
helm lint helm-charts/multi-env-app
```

---

### Issue #2: nginx Permission Errors (CrashLoopBackOff)
**Step:** 23-24 (Multi-Environment Helm Deployment)  
**Symptoms:**
- Pods in `CrashLoopBackOff` state
- Container logs show: `Permission denied` for `/var/cache/nginx/client_temp`
- Warning: `"user" directive makes sense only if the master process runs with super-user privileges`

**Root Cause:** Security context `runAsUser` conflicts with nginx's directory permissions

**Solution:**
Create custom nginx configuration using writable temp directories:

```yaml
# helm-charts/multi-env-app/templates/nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "multi-env-app.fullname" . }}-nginx-config
data:
  nginx.conf: |
    worker_processes auto;
    error_log /dev/stderr;
    pid /tmp/nginx.pid;
    
    events {
        worker_connections 1024;
    }
    
    http {
        client_body_temp_path /tmp/client_temp;
        proxy_temp_path       /tmp/proxy_temp_path;
        fastcgi_temp_path     /tmp/fastcgi_temp;
        uwsgi_temp_path       /tmp/uwsgi_temp;
        scgi_temp_path        /tmp/scgi_temp;
        
        server {
            listen 8080;
            location / {
                root /usr/share/nginx/html;
                index index.html;
            }
        }
    }
```

**Troubleshooting Commands:**
```powershell
# Check pod logs
kubectl logs -n <namespace> <pod-name>

# Check security context
kubectl describe pod -n <namespace> <pod-name>

# Test nginx config
kubectl exec -n <namespace> <pod-name> -- nginx -t
```

---

### Issue #3: ArgoCD Application Stuck in Progressing State
**Step:** 20-22 (Basic ArgoCD Setup)  
**Symptoms:**
- Application shows "Progressing" status indefinitely
- Resources appear to be created but application never becomes "Healthy"
- Sync operation completes but health check fails

**Root Cause:** Missing or incorrect health checks for custom resources

**Solution:**
Add proper health checks to ArgoCD application:

```yaml
# In ArgoCD Application spec
health:
  - group: apps
    kind: Deployment
    check: |
      hs = {}
      if obj.status ~= nil then
        if obj.status.readyReplicas == obj.status.replicas then
          hs.status = "Healthy"
        else
          hs.status = "Progressing"
        end
      end
      return hs
```

**Troubleshooting Commands:**
```powershell
# Check application status
kubectl describe application <app-name> -n argocd

# Check ArgoCD controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller

# Force refresh
argocd app sync <app-name>
```

---

### Issue #4: Git Repository Authentication Failures
**Step:** 20-21 (Personal Repository Setup)  
**Symptoms:**
- ArgoCD cannot access private GitHub repository
- Error: `authentication required` or `repository not found`
- Applications fail to sync with repository errors

**Root Cause:** Missing or incorrect Git credentials configuration

**Solution:**
Configure repository credentials in ArgoCD:

```powershell
# Add repository with credentials
argocd repo add https://github.com/username/repo --username <username> --password <token>

# Or via kubectl
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: repo-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/username/repo
  password: <personal-access-token>
  username: <username>
EOF
```

**Troubleshooting Commands:**
```powershell
# Test repository connectivity
argocd repo list

# Check repository credentials
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

---

## 🔐 Phase 2 Issues (Steps 27-31)

### Issue #5: Sealed Secrets Controller CrashLoopBackOff
**Step:** 27 (Sealed Secrets Installation)  
**Symptoms:**
- Sealed Secrets controller pod keeps restarting
- Error: `failed to create sealed secrets controller`
- CRD installation fails

**Root Cause:** Insufficient RBAC permissions or conflicting CRD versions

**Solution:**
1. Ensure CRDs are installed first:
```powershell
# Apply CRDs separately
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml
```

2. Check and fix RBAC permissions:
```yaml
# Ensure ClusterRole includes all necessary permissions
rules:
- apiGroups: [""]
  resources: ["secrets", "events"]
  verbs: ["create", "update", "delete", "get", "list", "watch", "patch"]
- apiGroups: ["bitnami.com"]
  resources: ["sealedsecrets"]
  verbs: ["get", "list", "watch", "update", "patch"]
```

**Troubleshooting Commands:**
```powershell
# Check controller logs
kubectl logs -n sealed-secrets-system -l app.kubernetes.io/name=sealed-secrets-controller

# Verify CRDs
kubectl get crd sealedsecrets.bitnami.com

# Check RBAC
kubectl auth can-i create secrets --as=system:serviceaccount:sealed-secrets-system:sealed-secrets-controller
```

---

### Issue #6: kubeseal CLI Tool Installation Failures
**Step:** 28 (kubeseal CLI Setup)  
**Symptoms:**
- kubeseal command not found
- Download fails or binary not executable
- Permission denied errors

**Root Cause:** Incorrect installation method or PATH configuration

**Solution:**
```powershell
# Alternative installation method using Chocolatey
choco install kubeseal

# Or manual installation with proper PATH
$kubesealVersion = "v0.24.0"
$url = "https://github.com/bitnami-labs/sealed-secrets/releases/download/$kubesealVersion/kubeseal-$kubesealVersion-windows-amd64.exe"
Invoke-WebRequest -Uri $url -OutFile "$env:USERPROFILE\tools\kubeseal.exe"

# Add to permanent PATH
$currentPath = [Environment]::GetEnvironmentVariable("PATH", "User")
[Environment]::SetEnvironmentVariable("PATH", "$currentPath;$env:USERPROFILE\tools", "User")
```

**Troubleshooting Commands:**
```powershell
# Verify installation
kubeseal --version

# Check PATH
$env:PATH -split ";"

# Test connectivity to sealed secrets controller
kubeseal --fetch-cert --controller-name=sealed-secrets-controller --controller-namespace=sealed-secrets-system
```

---

### Issue #7: External Secrets Operator Connection Failures
**Step:** 29-30 (ESO and Vault Setup)  
**Symptoms:**
- ExternalSecret stuck in "SecretSyncError" state
- Error: `connection refused` to Vault server
- Authentication failures with Vault

**Root Cause:** Network connectivity issues or incorrect Vault configuration

**Solution:**
1. Verify Vault connectivity:
```powershell
# Test Vault health endpoint
kubectl exec -n vault-system deployment/vault -- vault status

# Check service DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup vault-service.vault-system.svc.cluster.local
```

2. Fix Vault authentication:
```yaml
# Ensure vault token secret is correctly base64 encoded
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: external-secrets-demo
data:
  token: cm9vdC10b2tlbi0xMjM=  # base64 encoded root-token-123
```

**Troubleshooting Commands:**
```powershell
# Check ExternalSecret status
kubectl describe externalsecrets <external-secret-name> -n <namespace>

# Check ESO controller logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets

# Test Vault API directly
curl -H "X-Vault-Token: root-token-123" http://localhost:30200/v1/secret/data/database
```

---

### Issue #8: Vault Initialization Job Failures
**Step:** 30 (Vault Setup)  
**Symptoms:**
- Vault init job fails or times out
- Secrets not created in Vault
- Job shows "BackoffLimitExceeded"

**Root Cause:** Vault not ready when job starts or incorrect Vault API calls

**Solution:**
```yaml
# Improved init job with better wait logic
apiVersion: batch/v1
kind: Job
metadata:
  name: vault-init
spec:
  template:
    spec:
      containers:
      - name: vault-init
        image: hashicorp/vault:1.15.2
        command:
        - /bin/sh
        - -c
        - |
          # Better wait logic
          max_attempts=30
          attempt=0
          while [ $attempt -lt $max_attempts ]; do
            if vault status 2>/dev/null; then
              echo "Vault is ready!"
              break
            fi
            echo "Waiting for Vault... (attempt $((attempt + 1))/$max_attempts)"
            sleep 10
            attempt=$((attempt + 1))
          done
          
          if [ $attempt -eq $max_attempts ]; then
            echo "Vault failed to become ready"
            exit 1
          fi
          
          # Rest of initialization script...
```

**Troubleshooting Commands:**
```powershell
# Check job status
kubectl describe job vault-init -n vault-system

# Check job logs
kubectl logs job/vault-init -n vault-system

# Manual Vault initialization
kubectl exec -it deployment/vault -n vault-system -- vault auth -method=token token=root-token-123
```

---

### Issue #9: ArgoCD Application Health Check Failures
**Step:** 31 (External Secrets Demo)  
**Symptoms:**
- Application shows "Degraded" health status
- Custom resource health checks not working
- Resources created but application marked unhealthy

**Root Cause:** Missing or incorrect health check configuration for ExternalSecrets

**Solution:**
```yaml
# Add proper health check for ExternalSecrets in ArgoCD Application
health:
  - group: external-secrets.io
    kind: ExternalSecret
    check: |
      hs = {}
      if obj.status ~= nil and obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Ready" and condition.status == "True" then
            hs.status = "Healthy"
            hs.message = "ExternalSecret is ready"
            return hs
          elseif condition.type == "Ready" and condition.status == "False" then
            hs.status = "Degraded"
            hs.message = condition.message or "ExternalSecret not ready"
            return hs
          end
        end
      end
      hs.status = "Progressing"
      hs.message = "ExternalSecret status unknown"
      return hs
```

**Troubleshooting Commands:**
```powershell
# Check application health
kubectl get application <app-name> -n argocd -o jsonpath='{.status.health}'

# Check resource health in ArgoCD UI
# Navigate to http://localhost:30001

# Force health check refresh
argocd app sync <app-name> --force
```

---

## 🛠️ General Troubleshooting Commands

### ArgoCD Diagnostics
```powershell
# Check all ArgoCD applications
kubectl get applications -n argocd

# Check ArgoCD controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50

# Check ArgoCD server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=50

# Check ArgoCD repo server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=50
```

### Kubernetes Diagnostics
```powershell
# Check cluster status
kubectl cluster-info

# Check node status
kubectl get nodes

# Check all pods across namespaces
kubectl get pods --all-namespaces

# Check events for troubleshooting
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Git and Repository Issues
```powershell
# Check git status
git status

# Verify remote repository
git remote -v

# Check git credentials
git config --list

# Test repository connectivity
git ls-remote origin
```

---

## 📚 Best Practices for Troubleshooting

### 1. **Systematic Approach**
- Check logs first (pods, controllers, jobs)
- Verify resource status (`kubectl describe`)
- Check events (`kubectl get events`)
- Validate configurations (YAML syntax, values)

### 2. **Common Root Causes**
- **RBAC Issues**: Check service account permissions
- **Network Issues**: Verify service DNS and connectivity
- **Resource Constraints**: Check CPU/memory limits
- **Configuration Errors**: Validate YAML and values files

### 3. **Debugging Tools**
```powershell
# Debug pod for network testing
kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# Port forwarding for direct access
kubectl port-forward service/<service-name> 8080:80

# Execute commands in running pods
kubectl exec -it <pod-name> -- /bin/sh
```

### 4. **Prevention Strategies**
- **Version Pinning**: Always specify exact versions
- **Gradual Rollouts**: Test in staging before production
- **Monitoring**: Set up proper health checks and alerts
- **Documentation**: Keep track of configuration changes

---

## 🔄 Issue Resolution Workflow

1. **Identify the Problem**
   - Check application/pod status
   - Review error messages and logs
   - Determine the affected components

2. **Gather Information**
   - Collect logs from all related components
   - Check resource specifications
   - Verify network connectivity

3. **Analyze Root Cause**
   - Compare working vs. failing configurations
   - Check for recent changes
   - Validate dependencies

4. **Implement Solution**
   - Apply configuration fixes
   - Restart affected components
   - Verify resolution

5. **Verify and Monitor**
   - Test application functionality
   - Monitor for recurring issues
   - Document the solution

---

## 📞 Getting Help

### Community Resources
- **ArgoCD GitHub**: https://github.com/argoproj/argo-cd/issues
- **Sealed Secrets GitHub**: https://github.com/bitnami-labs/sealed-secrets/issues
- **External Secrets GitHub**: https://github.com/external-secrets/external-secrets/issues
- **Kubernetes Slack**: kubernetes.slack.com

### Documentation Links
- **ArgoCD Docs**: https://argo-cd.readthedocs.io/
- **Sealed Secrets Docs**: https://sealed-secrets.netlify.app/
- **External Secrets Docs**: https://external-secrets.io/
- **Kubernetes Docs**: https://kubernetes.io/docs/

---

**Last Updated:** June 29, 2025  
**Covers:** GitOps Advanced Learning Path Phase 1 (Steps 20-26) & Phase 2 (Steps 27-31)

*Keep this document updated as you encounter new issues during your GitOps journey!* 🚀
