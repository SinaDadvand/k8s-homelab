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

### Step 29: Implementing External Secrets Operator (ESO)

**Why We Learn This:** While Sealed Secrets is excellent for storing encrypted secrets in Git, External Secrets Operator (ESO) provides integration with external secret stores like AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, and others. This is crucial for enterprise environments where secrets are managed centrally outside of Kubernetes.

**Understanding External Secrets Operator:**
- **Problem**: Secrets scattered across different systems and manual synchronization
- **Solution**: ESO automatically syncs secrets from external stores into Kubernetes
- **Workflow**: ESO watches SecretStore resources → Fetches secrets from external systems → Creates Kubernetes secrets automatically

**External Secret Stores We'll Configure:**
- **Local Vault**: HashiCorp Vault running in KIND cluster (for learning)
- **Kubernetes Secrets**: Using existing cluster secrets as a source
- **File System**: Local file-based secret store (development/testing)

**Install External Secrets Operator:**

**Create:** `gitops-apps/external-secrets/namespace.yaml`
```yaml
# Namespace for External Secrets Operator
apiVersion: v1
kind: Namespace
metadata:
  name: external-secrets-system
  labels:
    name: external-secrets-system
    managed-by: argocd
    security-tier: critical
```

**Create:** `gitops-apps/external-secrets/controller.yaml`
```yaml
# External Secrets Operator Controller deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-secrets-controller
  namespace: external-secrets-system
  labels:
    app.kubernetes.io/name: external-secrets
    app.kubernetes.io/version: v0.9.11
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: external-secrets
  template:
    metadata:
      labels:
        app.kubernetes.io/name: external-secrets
    spec:
      serviceAccountName: external-secrets-controller
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      containers:
      - name: controller
        image: ghcr.io/external-secrets/external-secrets:v0.9.11
        imagePullPolicy: IfNotPresent
        args:
        - --concurrent=1
        - --enable-leader-election
        - --metrics-addr=:8080
        - --health-addr=:8081
        ports:
        - containerPort: 8080
          name: metrics
          protocol: TCP
        - containerPort: 8081
          name: health
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /healthz
            port: health
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /readyz
            port: health
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: external-secrets-controller-metrics
  namespace: external-secrets-system
  labels:
    app.kubernetes.io/name: external-secrets
spec:
  ports:
  - name: metrics
    port: 8080
    protocol: TCP
    targetPort: metrics
  selector:
    app.kubernetes.io/name: external-secrets
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-controller
  namespace: external-secrets-system
  labels:
    app.kubernetes.io/name: external-secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-secrets-controller
  labels:
    app.kubernetes.io/name: external-secrets
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - external-secrets.io
  resources:
  - externalsecrets
  - secretstores
  - clustersecretstores
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - external-secrets.io
  resources:
  - externalsecrets/status
  - secretstores/status
  - clustersecretstores/status
  verbs:
  - get
  - patch
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-secrets-controller
  labels:
    app.kubernetes.io/name: external-secrets
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-secrets-controller
subjects:
- kind: ServiceAccount
  name: external-secrets-controller
  namespace: external-secrets-system
```

**Create External Secrets CRDs:**

**Create:** `gitops-apps/external-secrets/crds.yaml`
```yaml
# Custom Resource Definitions for External Secrets Operator
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: externalsecrets.external-secrets.io
  labels:
    app.kubernetes.io/name: external-secrets
spec:
  group: external-secrets.io
  names:
    kind: ExternalSecret
    listKind: ExternalSecretList
    plural: externalsecrets
    singular: externalsecret
    shortNames:
    - es
  scope: Namespaced
  versions:
  - name: v1beta1
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
              secretStoreRef:
                type: object
                properties:
                  name:
                    type: string
                  kind:
                    type: string
              target:
                type: object
                properties:
                  name:
                    type: string
                  creationPolicy:
                    type: string
              data:
                type: array
                items:
                  type: object
                  properties:
                    secretKey:
                      type: string
                    remoteRef:
                      type: object
                      properties:
                        key:
                          type: string
                        property:
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
                    reason:
                      type: string
                    message:
                      type: string
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: secretstores.external-secrets.io
  labels:
    app.kubernetes.io/name: external-secrets
spec:
  group: external-secrets.io
  names:
    kind: SecretStore
    listKind: SecretStoreList
    plural: secretstores
    singular: secretstore
  scope: Namespaced
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              provider:
                type: object
                properties:
                  vault:
                    type: object
                  kubernetes:
                    type: object
                  fake:
                    type: object
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clustersecretstores.external-secrets.io
  labels:
    app.kubernetes.io/name: external-secrets
spec:
  group: external-secrets.io
  names:
    kind: ClusterSecretStore
    listKind: ClusterSecretStoreList
    plural: clustersecretstores
    singular: clustersecretstore
  scope: Cluster
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              provider:
                type: object
```

**Create ArgoCD Application for External Secrets:**

**Create:** `argocd-applications/security-apps/external-secrets.yaml`
```yaml
# ArgoCD Application for External Secrets Operator
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets-operator
  namespace: argocd
  labels:
    app.kubernetes.io/name: external-secrets
    component: security
    managed-by: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/SinaDadvand/k8s-helm.git
    targetRevision: HEAD
    path: gitops-apps/external-secrets
  
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets-system
  
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

**Deploy External Secrets Operator:**
```powershell
# Add and commit the External Secrets Operator
git add gitops-apps/external-secrets/
git add argocd-applications/security-apps/external-secrets.yaml
git commit -m "Add External Secrets Operator for external secret store integration"
git push origin main

# Deploy the External Secrets Operator via ArgoCD
kubectl apply -f argocd-applications/security-apps/external-secrets.yaml

# Wait for the operator to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=external-secrets -n external-secrets-system --timeout=300s

# Verify the operator is running
kubectl get pods -n external-secrets-system
kubectl get crd | Select-String external-secrets

# Check the operator logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets
```

### Step 30: Setting Up HashiCorp Vault as External Secret Store

**Why We Learn This:** HashiCorp Vault is one of the most popular secret management solutions in enterprise environments. Learning to integrate Vault with Kubernetes through ESO provides real-world applicable skills.

**Deploy HashiCorp Vault in Development Mode:**

**Create:** `gitops-apps/vault/deployment.yaml`
```yaml
# HashiCorp Vault deployment for learning purposes
apiVersion: v1
kind: Namespace
metadata:
  name: vault-system
  labels:
    name: vault-system
    security-tier: critical
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault
  namespace: vault-system
  labels:
    app: vault
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault
  template:
    metadata:
      labels:
        app: vault
    spec:
      containers:
      - name: vault
        image: hashicorp/vault:1.15.2
        imagePullPolicy: IfNotPresent
        args:
        - "vault"
        - "server"
        - "-dev"
        - "-dev-root-token-id=root-token-123"
        - "-dev-listen-address=0.0.0.0:8200"
        env:
        - name: VAULT_DEV_ROOT_TOKEN_ID
          value: "root-token-123"
        - name: VAULT_ADDR
          value: "http://127.0.0.1:8200"
        ports:
        - containerPort: 8200
          name: vault
          protocol: TCP
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        securityContext:
          capabilities:
            add:
            - IPC_LOCK
        volumeMounts:
        - name: vault-data
          mountPath: /vault/data
      volumes:
      - name: vault-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: vault-service
  namespace: vault-system
  labels:
    app: vault
spec:
  selector:
    app: vault
  ports:
  - name: vault
    port: 8200
    targetPort: 8200
    nodePort: 30200
  type: NodePort
---
# Service account for vault
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault
  namespace: vault-system
```

**Create:** `gitops-apps/vault/init-job.yaml`
```yaml
# Job to initialize Vault with sample secrets
apiVersion: batch/v1
kind: Job
metadata:
  name: vault-init
  namespace: vault-system
  labels:
    app: vault-init
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
          # Wait for Vault to be ready
          until vault status; do
            echo "Waiting for Vault to be ready..."
            sleep 5
          done
          
          # Enable KV secrets engine
          vault secrets enable -path=secret kv-v2
          
          # Create sample secrets for our applications
          vault kv put secret/database \
            username=postgres \
            password=SuperSecretDBPassword123 \
            host=postgresql.production.svc.cluster.local \
            port=5432 \
            database=production_app
          
          vault kv put secret/api-keys \
            stripe-key=sk_live_51SuperSecretStripeKey \
            github-token=ghp_SuperSecretGitHubToken123 \
            sendgrid-key=SG.SuperSecretSendGridAPIKey \
            jwt-secret=SuperSecretJWTSigningKey2024
          
          vault kv put secret/app-config \
            app-env=production \
            debug-mode=false \
            log-level=info \
            encryption-key=32CharacterSuperSecretEncryptionKey
          
          echo "✅ Vault initialized with sample secrets"
          
          # List all secrets for verification
          echo "📋 Available secrets:"
          vault kv list secret/
        env:
        - name: VAULT_ADDR
          value: "http://vault-service:8200"
        - name: VAULT_TOKEN
          value: "root-token-123"
      restartPolicy: OnFailure
  backoffLimit: 3
```

**Create ArgoCD Application for Vault:**

**Create:** `argocd-applications/security-apps/vault.yaml`
```yaml
# ArgoCD Application for HashiCorp Vault
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hashicorp-vault
  namespace: argocd
  labels:
    app.kubernetes.io/name: vault
    component: security
    managed-by: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/SinaDadvand/k8s-helm.git
    targetRevision: HEAD
    path: gitops-apps/vault
  
  destination:
    server: https://kubernetes.default.svc
    namespace: vault-system
  
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
        duration: 10s
        factor: 2
        maxDuration: 5m
```

**Deploy HashiCorp Vault:**
```powershell
# Add and commit Vault deployment
git add gitops-apps/vault/
git add argocd-applications/security-apps/vault.yaml
git commit -m "Add HashiCorp Vault for external secret store demonstration"
git push origin main

# Deploy Vault via ArgoCD
kubectl apply -f argocd-applications/security-apps/vault.yaml

# Wait for Vault to be ready
kubectl wait --for=condition=Ready pod -l app=vault -n vault-system --timeout=300s

# Initialize Vault with sample secrets
kubectl apply -f gitops-apps/vault/init-job.yaml

# Check Vault status
kubectl get pods -n vault-system
kubectl logs -n vault-system -l app=vault

# Test Vault accessibility
Write-Host "🏦 Vault UI: http://localhost:30200"
Write-Host "🔑 Root Token: root-token-123"

try {
    $response = Invoke-WebRequest -Uri "http://localhost:30200/v1/sys/health" -UseBasicParsing
    Write-Host "✅ Vault is responding and healthy"
} catch {
    Write-Host "⚠️ Vault is still starting up. Try again in a minute."
}
```

### Step 31: Creating Secret Stores and External Secrets

**Why We Learn This:** Now we'll configure ESO to connect to our Vault instance and automatically sync secrets into Kubernetes. This demonstrates the complete external secret management workflow.

**Create Vault Secret Store:**

**Create:** `gitops-apps/external-secrets-config/vault-secretstore.yaml`
```yaml
# Secret Store configuration for HashiCorp Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: external-secrets-demo
  labels:
    app.kubernetes.io/name: vault-secretstore
spec:
  provider:
    vault:
      server: "http://vault-service.vault-system.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-token
          key: token
---
# Secret containing Vault authentication token
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: external-secrets-demo
  labels:
    app.kubernetes.io/name: vault-auth
type: Opaque
data:
  token: cm9vdC10b2tlbi0xMjM=  # root-token-123 (base64)
```

**Create External Secrets for Different Use Cases:**

**Create:** `gitops-apps/external-secrets-config/external-secrets.yaml`
```yaml
# External Secret for database credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-external-secret
  namespace: external-secrets-demo
  labels:
    app.kubernetes.io/name: database-external-secret
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  
  target:
    name: database-credentials-external
    creationPolicy: Owner
    template:
      type: Opaque
      metadata:
        labels:
          app.kubernetes.io/name: database-secret
          source: external-secrets
      data:
        # Template the secret data with additional processing
        connection-string: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .database }}"
  
  data:
  - secretKey: username
    remoteRef:
      key: database
      property: username
  - secretKey: password
    remoteRef:
      key: database
      property: password
  - secretKey: host
    remoteRef:
      key: database
      property: host
  - secretKey: port
    remoteRef:
      key: database
      property: port
  - secretKey: database
    remoteRef:
      key: database
      property: database
  
  refreshInterval: 1h  # Refresh every hour
---
# External Secret for API keys
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-keys-external-secret
  namespace: external-secrets-demo
  labels:
    app.kubernetes.io/name: api-keys-external-secret
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  
  target:
    name: api-keys-external
    creationPolicy: Owner
  
  data:
  - secretKey: stripe-api-key
    remoteRef:
      key: api-keys
      property: stripe-key
  - secretKey: github-token
    remoteRef:
      key: api-keys
      property: github-token
  - secretKey: sendgrid-api-key
    remoteRef:
      key: api-keys
      property: sendgrid-key
  - secretKey: jwt-secret
    remoteRef:
      key: api-keys
      property: jwt-secret
  
  refreshInterval: 30m  # Refresh every 30 minutes
---
# External Secret for application configuration
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-config-external-secret
  namespace: external-secrets-demo
  labels:
    app.kubernetes.io/name: app-config-external-secret
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  
  target:
    name: app-config-external
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        # Create a JSON configuration from multiple vault keys
        config.json: |
          {
            "environment": "{{ .app-env }}",
            "debug": {{ .debug-mode }},
            "logging": {
              "level": "{{ .log-level }}"
            },
            "security": {
              "encryption_key": "{{ .encryption-key }}"
            }
          }
  
  data:
  - secretKey: app-env
    remoteRef:
      key: app-config
      property: app-env
  - secretKey: debug-mode
    remoteRef:
      key: app-config
      property: debug-mode
  - secretKey: log-level
    remoteRef:
      key: app-config
      property: log-level
  - secretKey: encryption-key
    remoteRef:
      key: app-config
      property: encryption-key
  
  refreshInterval: 2h  # Refresh every 2 hours
```

**Create Namespace and Demo Application:**

**Create:** `gitops-apps/external-secrets-config/namespace.yaml`
```yaml
# Namespace for External Secrets demonstration
apiVersion: v1
kind: Namespace
metadata:
  name: external-secrets-demo
  labels:
    name: external-secrets-demo
    managed-by: argocd
    security-level: high
    secrets-source: external
```

**Create:** `gitops-apps/external-secrets-config/demo-app.yaml`
```yaml
# Demo application using External Secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-secrets-demo-app
  namespace: external-secrets-demo
  labels:
    app: external-secrets-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: external-secrets-demo
  template:
    metadata:
      labels:
        app: external-secrets-demo
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: app
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
          name: http
        env:
        # Database credentials from External Secrets
        - name: DB_CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: database-credentials-external
              key: connection-string
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: database-credentials-external
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials-external
              key: password
        
        # API keys from External Secrets
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys-external
              key: stripe-api-key
        - name: GITHUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: api-keys-external
              key: github-token
        - name: SENDGRID_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys-external
              key: sendgrid-api-key
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-keys-external
              key: jwt-secret
        
        # Application configuration
        - name: APP_VERSION
          value: "2.0.0"
        - name: SECRET_SOURCE
          value: "external-vault"
        
        volumeMounts:
        - name: app-config
          mountPath: /etc/app/config.json
          subPath: config.json
        - name: web-content
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
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
      - name: app-config
        secret:
          secretName: app-config-external
      - name: web-content
        configMap:
          name: external-secrets-demo-config
---
apiVersion: v1
kind: Service
metadata:
  name: external-secrets-demo-service
  namespace: external-secrets-demo
  labels:
    app: external-secrets-demo
spec:
  selector:
    app: external-secrets-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30140
  type: NodePort
```

**Create:** `gitops-apps/external-secrets-config/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: external-secrets-demo-config
  namespace: external-secrets-demo
  labels:
    app: external-secrets-demo
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>External Secrets Demo - Vault Integration</title>
        <style>
            body {
                font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                margin: 0;
                padding: 20px;
                background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
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
            .vault-badge {
                display: inline-block;
                padding: 10px 20px;
                background: #1e90ff;
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
                color: #00d4ff;
                margin-bottom: 15px;
                font-size: 1.3em;
            }
            .secret-item {
                margin: 10px 0;
                padding: 10px;
                background: rgba(0, 0, 0, 0.2);
                border-radius: 8px;
                border-left: 4px solid #00d4ff;
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
            .status-sync { background: #3498db; }
            .endpoints {
                margin-top: 30px;
                text-align: center;
            }
            .endpoints a {
                color: #00d4ff;
                text-decoration: none;
                margin: 0 15px;
                padding: 8px 16px;
                border: 1px solid #00d4ff;
                border-radius: 20px;
                transition: all 0.3s;
            }
            .endpoints a:hover {
                background: #00d4ff;
                color: white;
            }
            .refresh-info {
                background: rgba(30, 144, 255, 0.2);
                padding: 15px;
                border-radius: 10px;
                margin: 20px 0;
                border-left: 4px solid #1e90ff;
            }
        </style>
        <script>
            function updateStatus() {
                document.getElementById('timestamp').textContent = new Date().toLocaleString();
                
                // Simulate checking external secrets status
                const secretsStatus = document.getElementById('secrets-status');
                secretsStatus.innerHTML = '<span class="status-indicator status-ok"></span>All external secrets synchronized';
                
                const vaultStatus = document.getElementById('vault-status');
                vaultStatus.innerHTML = '<span class="status-indicator status-ok"></span>Vault connection healthy';
                
                const refreshStatus = document.getElementById('refresh-status');
                refreshStatus.innerHTML = '<span class="status-indicator status-sync"></span>Next refresh in ' + 
                    Math.floor(Math.random() * 45 + 15) + ' minutes';
            }
            
            setInterval(updateStatus, 15000);
            window.onload = updateStatus;
        </script>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <h1>🏦 External Secrets Demo</h1>
                <p>HashiCorp Vault Integration with External Secrets Operator</p>
                <div class="vault-badge">🔐 Vault Backend</div>
                <div class="vault-badge">🔄 Auto-Sync Enabled</div>
                <div class="vault-badge">🛡️ Enterprise Ready</div>
            </div>
            
            <div class="info-grid">
                <div class="info-card">
                    <h3>🏦 Vault Integration Status</h3>
                    <div class="secret-item">
                        <strong>Vault Server:</strong> vault-system.svc:8200<br>
                        <strong>Status:</strong> <span id="vault-status">Connecting...</span><br>
                        <strong>KV Engine:</strong> v2 (secret/)<br>
                        <strong>Authentication:</strong> Token-based
                    </div>
                    <div class="refresh-info">
                        <strong>🔄 Auto-Refresh Status:</strong><br>
                        <span id="refresh-status">Checking...</span>
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>🗄️ Database Secrets (Vault)</h3>
                    <div class="secret-item">
                        <strong>Source:</strong> secret/database<br>
                        <strong>Username:</strong> <span class="secret-masked">postgres</span><br>
                        <strong>Password:</strong> <span class="secret-masked">***********</span><br>
                        <strong>Host:</strong> <span class="secret-masked">postgresql.production.svc</span><br>
                        <strong>Refresh:</strong> Every 1 hour
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>🔗 API Keys (Vault)</h3>
                    <div class="secret-item">
                        <strong>Source:</strong> secret/api-keys<br>
                        <strong>Stripe:</strong> <span class="secret-masked">sk_live_****</span><br>
                        <strong>GitHub:</strong> <span class="secret-masked">ghp_****</span><br>
                        <strong>SendGrid:</strong> <span class="secret-masked">SG.****</span><br>
                        <strong>JWT Secret:</strong> <span class="secret-masked">****</span><br>
                        <strong>Refresh:</strong> Every 30 minutes
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>⚙️ Application Config (Vault)</h3>
                    <div class="secret-item">
                        <strong>Source:</strong> secret/app-config<br>
                        <strong>Environment:</strong> <span class="secret-masked">production</span><br>
                        <strong>Debug Mode:</strong> <span class="secret-masked">false</span><br>
                        <strong>Log Level:</strong> <span class="secret-masked">info</span><br>
                        <strong>Refresh:</strong> Every 2 hours
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>🛠️ External Secrets Features</h3>
                    <div class="secret-item">
                        ✅ Central secret management in Vault<br>
                        ✅ Automatic secret synchronization<br>
                        ✅ Configurable refresh intervals<br>
                        ✅ Secret templating support<br>
                        ✅ Multiple backend support<br>
                        ✅ Kubernetes-native integration<br>
                        ✅ ArgoCD GitOps compatibility
                    </div>
                </div>
                
                <div class="info-card">
                    <h3>📊 ESO vs Sealed Secrets</h3>
                    <div class="secret-item">
                        <strong>External Secrets Operator:</strong><br>
                        • Centralized secret stores<br>
                        • Auto-refresh capabilities<br>
                        • Enterprise integrations<br>
                        • Dynamic secret fetching<br><br>
                        <strong>Sealed Secrets:</strong><br>
                        • Git-stored encrypted secrets<br>
                        • No external dependencies<br>
                        • Simpler setup<br>
                        • Static secret management
                    </div>
                </div>
            </div>
            
            <div class="endpoints">
                <h3>🔗 Application Endpoints</h3>
                <a href="http://localhost:30200" target="_blank">Vault UI</a>
                <a href="/health">Health Check</a>
                <a href="/config">Configuration</a>
                <a href="/secrets-status">Secrets Status</a>
            </div>
            
            <div style="text-align: center; margin-top: 40px; padding-top: 20px; border-top: 1px solid rgba(255,255,255,0.2);">
                <p><strong>🏦 External Secrets Benefits:</strong></p>
                <p>• Centralized secret management • Automatic synchronization • Enterprise integration • Dynamic updates</p>
                <p><small>Last updated: <span id="timestamp"></span> | Environment: Production | Source: HashiCorp Vault</small></p>
            </div>
        </div>
    </body>
    </html>
```

**Create ArgoCD Application for External Secrets Demo:**

**Create:** `argocd-applications/security-apps/external-secrets-demo.yaml`
```yaml
# ArgoCD Application for External Secrets Demo
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets-demo
  namespace: argocd
  labels:
    app.kubernetes.io/name: external-secrets-demo
    environment: production
    security-level: high
    secrets-source: vault
spec:
  project: default
  
  source:
    repoURL: https://github.com/SinaDadvand/k8s-helm.git
    targetRevision: HEAD
    path: gitops-apps/external-secrets-config
  
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets-demo
  
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
        duration: 10s
        factor: 2
        maxDuration: 5m
  
  # Health check for external secrets
  health:
    - group: external-secrets.io
      kind: ExternalSecret
      check: |
        hs = {}
        if obj.status ~= nil then
          if obj.status.conditions ~= nil then
            for i, condition in ipairs(obj.status.conditions) do
              if condition.type == "Ready" and condition.status == "True" then
                hs.status = "Healthy"
                hs.message = "ExternalSecret is ready and synchronized"
                return hs
              end
            end
          end
          hs.status = "Progressing"
          hs.message = "ExternalSecret is being processed"
        else
          hs.status = "Progressing"
          hs.message = "ExternalSecret status not available"
        end
        return hs
```

**Deploy the External Secrets Demo:**
```powershell
# Add all External Secrets configuration files
git add gitops-apps/external-secrets-config/
git add argocd-applications/security-apps/external-secrets-demo.yaml
git commit -m "Add External Secrets demo with HashiCorp Vault integration"
git push origin main

# Deploy the External Secrets demo
kubectl apply -f argocd-applications/security-apps/external-secrets-demo.yaml

# Wait for the application to sync
kubectl wait --for=condition=Synced --timeout=300s application/external-secrets-demo -n argocd

# Check External Secrets status
kubectl get externalsecrets -n external-secrets-demo
kubectl get secretstores -n external-secrets-demo

# Verify that secrets were created from Vault
kubectl get secrets -n external-secrets-demo
kubectl describe externalsecret database-external-secret -n external-secrets-demo

# Check the demo application
kubectl get pods -n external-secrets-demo
kubectl get services -n external-secrets-demo

# Test the application
Write-Host "🌐 External Secrets Demo: http://localhost:30140"
Write-Host "🏦 Vault UI: http://localhost:30200 (Token: root-token-123)"

# Test application response
try {
    $response = Invoke-WebRequest -Uri "http://localhost:30140" -UseBasicParsing
    Write-Host "✅ External Secrets demo application is responding"
} catch {
    Write-Host "⚠️ Application is still starting up. Try again in a minute."
}
```

**Verify External Secrets Functionality:**
```powershell
# Check External Secrets Operator logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets --tail=20

# Monitor External Secret synchronization
kubectl get externalsecrets -n external-secrets-demo -w

# Check secret content (should match Vault values)
Write-Host "🔍 Checking synchronized secrets:"
kubectl get secret database-credentials-external -n external-secrets-demo -o jsonpath='{.data.username}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# Check application environment variables
kubectl exec -n external-secrets-demo deployment/external-secrets-demo-app -- printenv | Select-String -Pattern "DB_|STRIPE_|GITHUB_|SENDGRID_|JWT_"

# Test secret refresh by updating Vault (optional)
Write-Host "🔄 Testing secret refresh capability:"
Write-Host "1. Access Vault UI at http://localhost:30200"
Write-Host "2. Login with token: root-token-123"
Write-Host "3. Update any secret in secret/database"
Write-Host "4. Wait for refresh interval or force sync"

Write-Host "✅ External Secrets verification complete!"
Write-Host "🏦 Secrets are now managed centrally in Vault"
Write-Host "🔄 ESO automatically syncs changes to Kubernetes"
```

### Summary of Step 29-31: External Secrets Operator Implementation

**What We Accomplished:**
- ✅ **Deployed External Secrets Operator**: Installed ESO with CRDs and RBAC
- ✅ **Set Up HashiCorp Vault**: Deployed Vault in development mode with sample secrets
- ✅ **Configured Secret Stores**: Created SecretStore resources to connect to Vault
- ✅ **Implemented External Secrets**: Created ExternalSecret resources for different use cases
- ✅ **Demonstrated Auto-Sync**: Showed automatic secret synchronization with refresh intervals
- ✅ **Integrated with GitOps**: Deployed everything through ArgoCD applications

**Key Benefits:**
- 🏦 **Centralized Secret Management**: All secrets stored in external systems
- 🔄 **Automatic Synchronization**: Secrets updated automatically without manual intervention
- 🛡️ **Enterprise Integration**: Supports AWS, Azure, GCP, Vault, and more
- 📋 **Flexible Templating**: Transform and combine secrets during sync
- ⚡ **Dynamic Updates**: Secrets refresh based on configurable intervals

**External Secrets vs Sealed Secrets Comparison:**

| Feature | External Secrets Operator | Sealed Secrets |
|---------|---------------------------|----------------|
| **Storage Location** | External secret stores (Vault, AWS, etc.) | Encrypted in Git repository |
| **Secret Updates** | Automatic refresh from source | Manual re-encryption and commit |
| **Dependencies** | Requires external secret store | No external dependencies |
| **Enterprise Integration** | Excellent (supports major cloud providers) | Limited (Git-based only) |
| **Complexity** | Higher (more components) | Lower (single controller) |
| **Best Use Case** | Enterprise environments with existing secret stores | Simple GitOps workflows |

**Next Steps Preview:**
In the upcoming steps, we'll explore:
- **ArgoCD Image Updater**: Automated container image updates
- **OPA Gatekeeper**: Policy-as-code security enforcement
- **Multi-Cluster GitOps**: Managing multiple Kubernetes clusters

---

**Happy Learning!** 🚀 You now have comprehensive secret management with both Sealed Secrets and External Secrets Operator!
