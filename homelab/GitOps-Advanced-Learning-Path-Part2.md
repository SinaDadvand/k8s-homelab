# Kubernetes GitOps Advanced Learning Path - Part 2
**Production-Ready GitOps with Enhanced Security, Automation, and Monitoring**

## Prerequisites
- ✅ Completed GitOps Advanced Learning Path Part 1 (Steps 20-26)
- ✅ KIND cluster running with ArgoCD installed and configured
- ✅ Personal GitHub repository with GitOps workflows established
- ✅ Multiple applications deployed using App of Apps pattern
- ✅ Understanding of Helm charts and ArgoCD sync strategies
- ✅ ArgoCD Web UI accessible at http://localhost:30001

## Repository Structure Enhancement
Building upon your existing repository structure, we'll add new components:
```
your-repo/
├── helm-charts/
│   ├── multi-env-app/
│   └── monitoring-stack/     # New: Monitoring components
├── gitops-apps/
│   ├── nginx-app/
│   ├── redis-app/
│   ├── sealed-secrets/       # New: Sealed Secrets controller
│   ├── external-secrets/     # New: External Secrets Operator
│   └── security/             # New: Security policies
├── argocd-applications/
│   ├── app-of-apps.yaml
│   ├── individual-apps/
│   └── security-apps/        # New: Security-focused applications
├── secrets/
│   ├── sealed/               # New: Sealed secrets
│   └── templates/            # New: Secret templates
└── policies/                 # New: OPA Gatekeeper policies
    ├── security/
    └── compliance/
```

---

## Phase 2: Production-Ready GitOps Patterns

### Step 27: Implementing Sealed Secrets for Secure Secret Management

**Why We Learn This:** Storing secrets in Git repositories is a major security risk. Sealed Secrets provides a way to encrypt secrets that can only be decrypted by the Sealed Secrets controller running in your cluster. This allows you to store encrypted secrets in Git while maintaining GitOps principles.

**Understanding Sealed Secrets:**
- **Problem**: Regular Kubernetes secrets are base64 encoded (not encrypted)
- **Solution**: Sealed Secrets encrypts secrets using public key cryptography
- **Workflow**: You encrypt secrets locally → store encrypted secrets in Git → Sealed Secrets controller decrypts them in cluster

**Install Sealed Secrets Controller:**

**Create:** `gitops-apps/sealed-secrets/controller.yaml`
```yaml
# Sealed Secrets Controller deployment
apiVersion: v1
kind: Namespace
metadata:
  name: sealed-secrets-system
  labels:
    name: sealed-secrets-system
    managed-by: argocd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sealed-secrets-controller
  namespace: sealed-secrets-system
  labels:
    app.kubernetes.io/name: sealed-secrets-controller
    app.kubernetes.io/version: v0.24.0
spec:
  minReadySeconds: 30
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: sealed-secrets-controller
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: sealed-secrets-controller
        app.kubernetes.io/version: v0.24.0
    spec:
      containers:
      - args:
        - --update-status
        - --rotation-period=720h
        - --key-prefix=sealed-secrets-key
        command:
        - controller
        env:
        - name: SEALED_SECRETS_UPDATE_STATUS
          value: "true"
        image: docker.io/bitnami/sealed-secrets-controller:v0.24.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
        name: sealed-secrets-controller
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 8081
          name: metrics
          protocol: TCP
        readinessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1001
        volumeMounts:
        - mountPath: /tmp
          name: tmp
      securityContext:
        fsGroup: 65534
      serviceAccountName: sealed-secrets-controller
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: tmp
---
apiVersion: v1
kind: Service
metadata:
  name: sealed-secrets-controller
  namespace: sealed-secrets-system
  labels:
    app.kubernetes.io/name: sealed-secrets-controller
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: http
  - name: metrics
    port: 8081
    protocol: TCP
    targetPort: metrics
  selector:
    app.kubernetes.io/name: sealed-secrets-controller
  type: ClusterIP
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sealed-secrets-controller
  namespace: sealed-secrets-system
  labels:
    app.kubernetes.io/name: sealed-secrets-controller
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sealed-secrets-controller
  labels:
    app.kubernetes.io/name: sealed-secrets-controller
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - create
  - update
  - delete
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - bitnami.com
  resources:
  - sealedsecrets
  verbs:
  - get
  - list
  - watch
  - update
  - patch
- apiGroups:
  - bitnami.com
  resources:
  - sealedsecrets/status
  verbs:
  - update
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sealed-secrets-controller
  labels:
    app.kubernetes.io/name: sealed-secrets-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: sealed-secrets-controller
subjects:
- kind: ServiceAccount
  name: sealed-secrets-controller
  namespace: sealed-secrets-system
---
# Custom Resource Definition for SealedSecrets
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: sealedsecrets.bitnami.com
  labels:
    app.kubernetes.io/name: sealed-secrets-controller
spec:
  group: bitnami.com
  names:
    kind: SealedSecret
    listKind: SealedSecretList
    plural: sealedsecrets
    singular: sealedsecret
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              encryptedData:
                type: object
                additionalProperties:
                  type: string
              template:
                type: object
                properties:
                  metadata:
                    type: object
                    properties:
                      name:
                        type: string
                      namespace:
                        type: string
                      labels:
                        type: object
                        additionalProperties:
                          type: string
                      annotations:
                        type: object
                        additionalProperties:
                          type: string
                  type:
                    type: string
          status:
            type: object
            properties:
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                    lastUpdateTime:
                      type: string
                    lastTransitionTime:
                      type: string
                    reason:
                      type: string
                    message:
                      type: string
```

**Create ArgoCD Application for Sealed Secrets:**

**Create:** `argocd-applications/security-apps/sealed-secrets.yaml`
```yaml
# ArgoCD Application for Sealed Secrets Controller
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets-controller
  namespace: argocd
  labels:
    app.kubernetes.io/name: sealed-secrets-controller
    component: security
    managed-by: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/SinaDadvand/k8s-helm.git
    targetRevision: HEAD
    path: gitops-apps/sealed-secrets
  
  destination:
    server: https://kubernetes.default.svc
    namespace: sealed-secrets-system
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Deploy Sealed Secrets Controller:**
```powershell
# Create the security-apps directory structure
New-Item -ItemType Directory -Path "argocd-applications\security-apps" -Force

# Add and commit the Sealed Secrets controller
git add gitops-apps/sealed-secrets/
git add argocd-applications/security-apps/sealed-secrets.yaml
git commit -m "Add Sealed Secrets controller for secure secret management"
git push origin main

# Deploy the Sealed Secrets controller via ArgoCD
kubectl apply -f argocd-applications/security-apps/sealed-secrets.yaml

# Wait for the controller to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=sealed-secrets-controller -n sealed-secrets-system --timeout=300s

# Verify the controller is running
kubectl get pods -n sealed-secrets-system
kubectl get services -n sealed-secrets-system

# Check the controller logs
kubectl logs -n sealed-secrets-system -l app.kubernetes.io/name=sealed-secrets-controller
```

**Install kubeseal CLI Tool:**
```powershell
# Download kubeseal CLI for Windows
$kubesealVersion = "v0.24.0"
$url = "https://github.com/bitnami-labs/sealed-secrets/releases/download/$kubesealVersion/kubeseal-$kubesealVersion-windows-amd64.tar.gz"

# Create tools directory
New-Item -ItemType Directory -Path "$env:USERPROFILE\tools" -Force

# Download and extract kubeseal
Invoke-WebRequest -Uri $url -OutFile "$env:USERPROFILE\tools\kubeseal.tar.gz"

# Extract the archive (you may need to install tar or use 7-Zip)
# For this example, we'll use the PowerShell method
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$env:USERPROFILE\tools\kubeseal.tar.gz", "$env:USERPROFILE\tools\")

# Add to PATH (for current session)
$env:PATH += ";$env:USERPROFILE\tools"

# Verify installation
kubeseal --version

Write-Host "✅ kubeseal CLI installed successfully"
Write-Host "💡 Add $env:USERPROFILE\tools to your permanent PATH for future sessions"
```

**Alternative: Use Chocolatey to install kubeseal:**
```powershell
# If you have Chocolatey installed
choco install kubeseal

# Or use winget
winget install Bitnami.SealedSecrets
```

### Step 28: Creating and Managing Sealed Secrets

**Why We Learn This:** Now that we have the Sealed Secrets controller running, we'll learn how to create encrypted secrets that can be safely stored in Git repositories while maintaining the security of sensitive data.

**Fetch the Public Key:**
```powershell
# Get the public key from the Sealed Secrets controller
kubeseal --fetch-cert --controller-name=sealed-secrets-controller --controller-namespace=sealed-secrets-system > sealed-secrets-public.pem

# Verify the public key was fetched
Get-Content sealed-secrets-public.pem

Write-Host "✅ Public key fetched and saved to sealed-secrets-public.pem"
Write-Host "📝 This public key can be shared and used to encrypt secrets"
```

**Create Regular Secrets First:**

**Create:** `secrets/templates/database-secret.yaml`
```yaml
# Regular Kubernetes secret template (DO NOT commit this to Git)
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production-app
type: Opaque
data:
  username: YWRtaW4=  # admin (base64)
  password: UGFzc3cwcmQxMjM=  # Passw0rd123 (base64)
  host: cG9zdGdyZXNxbC5wcm9kdWN0aW9uLnN2Yy5jbHVzdGVyLmxvY2Fs  # postgresql.production.svc.cluster.local
  port: NTQzMg==  # 5432
  database: cHJvZHVjdGlvbl9kYg==  # production_db
```

**Create:** `secrets/templates/api-keys-secret.yaml`
```yaml
# API keys secret template (DO NOT commit this to Git)
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  namespace: production-app
type: Opaque
data:
  stripe-api-key: c2tfdGVzdF8xMjM0NTY3ODkwYWJjZGVm  # sk_test_1234567890abcdef
  github-token: Z2hwXzF6WjlCeFhYWFhYWFhYWFhYWFhYWFhYWFhYWA==  # ghp_1zZ9BxXXXXXXXXXXXXXXXXXXXXXX
  sendgrid-api-key: U0cuWVhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhY  # SG.YXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  jwt-secret: bXlTdXBlclNlY3JldEp3dFRva2VuS2V5MTIzNDU2Nzg=  # mySuperSecretJwtTokenKey12345678
```

**Encrypt Secrets with kubeseal:**
```powershell
# Create the sealed secrets directory
New-Item -ItemType Directory -Path "secrets\sealed" -Force

# Encrypt the database secret
kubeseal --format=yaml --cert=sealed-secrets-public.pem < secrets/templates/database-secret.yaml > secrets/sealed/database-secret-sealed.yaml

# Encrypt the API keys secret
kubeseal --format=yaml --cert=sealed-secrets-public.pem < secrets/templates/api-keys-secret.yaml > secrets/sealed/api-keys-sealed.yaml

# Verify the encrypted secrets
Write-Host "📁 Database Secret (Sealed):"
Get-Content secrets/sealed/database-secret-sealed.yaml | Select-Object -First 20

Write-Host "`n📁 API Keys Secret (Sealed):"
Get-Content secrets/sealed/api-keys-sealed.yaml | Select-Object -First 20

Write-Host "`n✅ Secrets encrypted successfully!"
Write-Host "🔒 These encrypted secrets are safe to commit to Git"
```

**Create GitOps Application Structure for Secrets:**

**Create:** `gitops-apps/production-app/namespace.yaml`
```yaml
# Namespace for production application
apiVersion: v1
kind: Namespace
metadata:
  name: production-app
  labels:
    name: production-app
    environment: production
    managed-by: argocd
    security-level: high
```

**Move Sealed Secrets to GitOps Structure:**
```powershell
# Create production app directory
New-Item -ItemType Directory -Path "gitops-apps\production-app" -Force

# Copy sealed secrets to the production app directory
Copy-Item "secrets\sealed\database-secret-sealed.yaml" "gitops-apps\production-app\"
Copy-Item "secrets\sealed\api-keys-sealed.yaml" "gitops-apps\production-app\"

# Verify the structure
Get-ChildItem "gitops-apps\production-app\" -Recurse
```

**Create a Test Application that Uses Sealed Secrets:**

**Create:** `gitops-apps/production-app/deployment.yaml`
```yaml
# Production application that uses sealed secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production-app
  labels:
    app: production-app
    version: v1.0.0
spec:
  replicas: 2
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: app
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
          name: http
        - containerPort: 8080
          name: metrics
        env:
        # Database connection using sealed secrets
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: host
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: port
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: database
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password
        # API keys using sealed secrets
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: stripe-api-key
        - name: GITHUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: github-token
        - name: SENDGRID_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: sendgrid-api-key
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: jwt-secret
        # Application configuration
        - name: APP_ENV
          value: "production"
        - name: APP_VERSION
          value: "1.0.0"
        volumeMounts:
        - name: config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      volumes:
      - name: config
        configMap:
          name: production-app-config
---
apiVersion: v1
kind: Service
metadata:
  name: production-app-service
  namespace: production-app
  labels:
    app: production-app
spec:
  selector:
    app: production-app
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30130
  - name: metrics
    port: 8080
    targetPort: 8080
  type: NodePort
```

**Create:** `gitops-apps/production-app/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: production-app-config
  namespace: production-app
  labels:
    app: production-app
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Production App - Sealed Secrets Demo</title>
        <style>
            body {
                font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                margin: 0;
                padding: 20px;
                background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
                color: white;
                min-height: 100vh;
            }
            .container {
                max-width: 1200px;
                margin: 0 auto;
                background: rgba(255, 255, 255, 0.1);
                padding: 40px;
                border-radius: 15px;
                backdrop-filter: blur(10px);
                box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
            }
            .header {
                text-align: center;
                margin-bottom: 40px;
            }
            .header h1 {
                font-size: 3em;
                margin-bottom: 10px;
                background: linear-gradient(45deg, #fff, #f0f0f0);
                -webkit-background-clip: text;
                -webkit-text-fill-color: transparent;
                background-clip: text;
            }
            .security-badge {
                display: inline-block;
                padding: 10px 20px;
                background: #27ae60;
                border-radius: 25px;
                font-weight: bold;
                margin: 10px 5px;
            }
            .info-grid {
                display: grid;
                grid-template-columns: repeat(auto-fit, minmax(350px, 1fr));
                gap: 20px;
                margin-top: 30px;
            }
            .info-card {
                background: rgba(0, 0, 0, 0.3);
                padding: 25px;
                border-radius: 15px;
                border: 1px solid rgba(255, 255, 255, 0.2);
            }
            .info-card h3 {
                color: #3498db;
                margin-bottom: 15px;
                font-size: 1.3em;
            }
            .secret-item {
                margin: 10px 0;
                padding: 10px;
                background: rgba(0, 0, 0, 0.2);
                border-radius: 8px;
                border-left: 4px solid #e74c3c;
            }
            .secret-masked {
                font-family: 'Courier New', monospace;
                background: #2c3e50;
                padding: 3px 8px;
                border-radius: 4px;
                color: #ecf0f1;
            }
            .status-indicator {
                display: inline-block;
                width: 12px;
                height: 12px;
                border-radius: 50%;
                margin-right: 8px;
            }
            .status-ok { background: #2ecc71; }
            .status-warning { background: #f39c12; }
            .endpoints {
                margin-top: 30px;
                text-align: center;
            }
            .endpoints a {
                color: #3498db;
                text-decoration: none;
                margin: 0 15px;
                padding: 8px 16px;
                border: 1px solid #3498db;
                border-radius: 20px;
                transition: all 0.3s;
            }
            .endpoints a:hover {
                background: #3498db;
                color: white;
            }
        </style>
        <script>
            function updateStatus() {
                document.getElementById('timestamp').textContent = new Date().toLocaleString();
                
                // Simulate checking sealed secrets status
                const secretsStatus = document.getElementById('secrets-status');
                secretsStatus.innerHTML = '<span class="status-indicator status-ok"></span>All secrets decrypted successfully';
                
                // Update connection status
                const dbStatus = document.getElementById('db-status');
                dbStatus.innerHTML = '<span class="status-indicator status-ok"></span>Database connection established';
                
                const apiStatus = document.getElementById('api-status');
                apiStatus.innerHTML = '<span class="status-indicator status-ok"></span>API keys validated';
            }
            
            setInterval(updateStatus, 10000);
            window.onload = updateStatus;
        </script>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <h1>🔐 Production Application</h1>
                <p>Demonstrating Sealed Secrets Integration with GitOps</p>
                <div class="security-badge">🛡️ Sealed Secrets Enabled</div>
                <div class="security-badge">🔒 GitOps Secure</div>
            </div>
            
            <div class="info-grid">
                <div class="info-card">
                    <h3>🔑 Sealed Secrets Status</h3>
                    <div class="secret-item">
                        <strong>Controller Status:</strong><br>
                        <span id="secrets-status">Loading...</span>
                    </div>
                    <div class="secret-item">
                        <strong>Encryption:</strong> RSA-2048<br>
                        <strong>Namespace Scoped:</strong> Yes<br>
                        <strong>Auto-rotation:</strong> 30 days
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>🗄️ Database Configuration</h3>
                    <div class="secret-item">
                        <strong>Host:</strong> <span class="secret-masked">*****.production.svc</span><br>
                        <strong>Database:</strong> <span class="secret-masked">production_**</span><br>
                        <strong>Username:</strong> <span class="secret-masked">*****</span><br>
                        <strong>Status:</strong> <span id="db-status">Checking...</span>
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>🔗 API Integrations</h3>
                    <div class="secret-item">
                        <strong>Stripe API:</strong> <span class="secret-masked">sk_****</span><br>
                        <strong>GitHub Token:</strong> <span class="secret-masked">ghp_****</span><br>
                        <strong>SendGrid:</strong> <span class="secret-masked">SG.****</span><br>
                        <strong>JWT Secret:</strong> <span class="secret-masked">****</span><br>
                        <strong>Status:</strong> <span id="api-status">Validating...</span>
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>🛠️ GitOps Security Features</h3>
                    <div class="secret-item">
                        ✅ Secrets encrypted at rest in Git<br>
                        ✅ Public key encryption (RSA)<br>
                        ✅ Namespace-scoped decryption<br>
                        ✅ Automatic secret rotation<br>
                        ✅ ArgoCD integration<br>
                        ✅ Audit trail maintained
                    </div>
                </div>
            </div>
            
            <div class="endpoints">
                <h3>🔗 Application Endpoints</h3>
                <a href="/health">Health Check</a>
                <a href="/metrics">Metrics</a>
                <a href="/config">Configuration</a>
                <a href="/secrets-status">Secrets Status</a>
            </div>
            
            <div style="text-align: center; margin-top: 40px; padding-top: 20px; border-top: 1px solid rgba(255,255,255,0.2);">
                <p><strong>🔐 Sealed Secrets Benefits:</strong></p>
                <p>• Store encrypted secrets safely in Git • Maintain GitOps principles • Automatic decryption in cluster • Key rotation support</p>
                <p><small>Last updated: <span id="timestamp"></span> | Environment: Production | Security: High</small></p>
            </div>
        </div>
    </body>
    </html>
```

**Create ArgoCD Application for Production App:**

**Create:** `argocd-applications/security-apps/production-app.yaml`
```yaml
# ArgoCD Application for Production App with Sealed Secrets
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app-sealed-secrets
  namespace: argocd
  labels:
    app.kubernetes.io/name: production-app
    environment: production
    security-level: high
    secrets-type: sealed
spec:
  project: default
  
  source:
    repoURL: https://github.com/SinaDadvand/k8s-helm.git
    targetRevision: HEAD
    path: gitops-apps/production-app
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production-app
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    
    # Sync waves for secrets first
    syncOptions:
    - SyncWave=0  # Secrets first
    
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 5m
  
  # Health check for sealed secrets
  health:
    - group: bitnami.com
      kind: SealedSecret
      check: |
        hs = {}
        if obj.status ~= nil then
          if obj.status.conditions ~= nil then
            for i, condition in ipairs(obj.status.conditions) do
              if condition.type == "Synced" and condition.status == "True" then
                hs.status = "Healthy"
                hs.message = "SealedSecret is synced"
                return hs
              end
            end
          end
          hs.status = "Progressing"
          hs.message = "SealedSecret is being processed"
        else
          hs.status = "Progressing"
          hs.message = "SealedSecret status not available"
        end
        return hs
```

**Deploy the Production Application with Sealed Secrets:**
```powershell
# Add all files to git
git add gitops-apps/production-app/
git add argocd-applications/security-apps/production-app.yaml
git add secrets/sealed/
git commit -m "Add production application with Sealed Secrets integration"
git push origin main

# Deploy the production application
kubectl apply -f argocd-applications/security-apps/production-app.yaml

# Wait for the application to sync
kubectl wait --for=condition=Synced --timeout=300s application/production-app-sealed-secrets -n argocd

# Verify sealed secrets were created and decrypted
kubectl get sealedsecrets -n production-app
kubectl get secrets -n production-app

# Check that regular secrets were created from sealed secrets
kubectl describe secret database-credentials -n production-app
kubectl describe secret api-keys -n production-app

# Verify the application is running
kubectl get pods -n production-app
kubectl get services -n production-app

# Test the application
Write-Host "🌐 Production App URL: http://localhost:30130"
Write-Host "🔐 Application using Sealed Secrets for secure secret management"

# Test application endpoints
try {
    $response = Invoke-WebRequest -Uri "http://localhost:30130" -UseBasicParsing
    Write-Host "✅ Application is responding"
} catch {
    Write-Host "⚠️ Application is still starting up. Try again in a minute."
}
```

**Verify Sealed Secrets Functionality:**
```powershell
# Check sealed secrets controller logs
kubectl logs -n sealed-secrets-system -l app.kubernetes.io/name=sealed-secrets-controller --tail=20

# Verify that sealed secrets were processed
kubectl get events -n production-app --field-selector reason=SuccessfulCreate

# Check that the secrets contain the correct keys
kubectl get secret database-credentials -n production-app -o jsonpath='{.data}' | ConvertFrom-Json

# Verify environment variables are set in the pods
kubectl exec -n production-app deployment/production-app -- printenv | Select-String -Pattern "DB_|STRIPE_|GITHUB_|SENDGRID_|JWT_"

Write-Host "✅ Sealed Secrets verification complete!"
Write-Host "🔒 Secrets are encrypted in Git but decrypted in the cluster"
Write-Host "📋 Check ArgoCD UI for application status: http://localhost:30001"
```

**Test Secret Rotation:**
```powershell
# Create a new secret with updated values
$newSecret = @"
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production-app
type: Opaque
data:
  username: YWRtaW4=  # admin
  password: TmV3UGFzc3cwcmQxMjM=  # NewPassw0rd123 (updated)
  host: cG9zdGdyZXNxbC5wcm9kdWN0aW9uLnN2Yy5jbHVzdGVyLmxvY2Fs
  port: NTQzMg==
  database: cHJvZHVjdGlvbl9kYg==
"@

# Save to temporary file
$newSecret | Out-File -FilePath "temp-secret.yaml" -Encoding UTF8

# Encrypt the updated secret
kubeseal --format=yaml --cert=sealed-secrets-public.pem < temp-secret.yaml > gitops-apps/production-app/database-secret-sealed.yaml

# Clean up temporary file
Remove-Item "temp-secret.yaml"

# Commit and push the updated sealed secret
git add gitops-apps/production-app/database-secret-sealed.yaml
git commit -m "Rotate database password using Sealed Secrets"
git push origin main

# ArgoCD will automatically detect and sync the new secret
Write-Host "🔄 Secret rotation initiated via GitOps"
Write-Host "⏳ ArgoCD will automatically sync the updated secret"

# Monitor the sync
kubectl get application production-app-sealed-secrets -n argocd -w
```

**Clean Up (Optional):**
```powershell
# Remove template secrets (they should never be committed to Git)
Remove-Item "secrets/templates/*" -Force
Write-Host "🧹 Template secrets removed (security best practice)"
Write-Host "✅ Only encrypted sealed secrets remain in Git repository"
```

### Summary of Step 27-28: Sealed Secrets Implementation

**What We Accomplished:**
- ✅ **Deployed Sealed Secrets Controller**: Installed and configured the controller via GitOps
- ✅ **Set Up Encryption Workflow**: Fetched public key and installed kubeseal CLI
- ✅ **Created Encrypted Secrets**: Encrypted sensitive data that can be safely stored in Git
- ✅ **Integrated with GitOps**: Deployed applications that use sealed secrets automatically
- ✅ **Implemented Security Best Practices**: Namespace scoping, key rotation, audit trails
- ✅ **Tested Secret Rotation**: Demonstrated how to update secrets through GitOps

**Key Benefits:**
- 🔒 **Git-Safe Secrets**: Store encrypted secrets in Git repositories
- 🔄 **GitOps Compatible**: Maintain declarative configuration for secrets
- 🛡️ **Enhanced Security**: Public key encryption with automatic decryption
- 📋 **Audit Trail**: All secret changes tracked in Git history
- ⚡ **Automatic Sync**: ArgoCD handles secret deployment and updates

**Next Steps Preview:**
In the upcoming steps, we'll explore:
- **External Secrets Operator**: Integration with external secret stores
- **ArgoCD Image Updater**: Automated container image updates
- **OPA Gatekeeper**: Policy-as-code security enforcement

---

**Happy Learning!** 🚀 You now have a secure, GitOps-compatible secret management system using Sealed Secrets!
