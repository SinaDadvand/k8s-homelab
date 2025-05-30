# Kubernetes Beginner Learning Path
**Hands-on Learning with KIND Cluster**

## Prerequisites
- ✅ KIND installed and configured
- ✅ Docker Desktop running
- ✅ kubectl installed
- ✅ KIND cluster configuration ready (`kind-cluster.yaml`)

## Overview
This learning path will take you through essential Kubernetes concepts using practical, hands-on examples. We'll use three key applications:
1. **Nginx** - Web server for learning core concepts
2. **Busybox** - Debugging and troubleshooting tool
3. **Redis** - Database for persistent storage concepts

---

## Phase 1: Cluster Setup and Basic Operations

### Step 1: Create Your KIND Cluster
```powershell
# Navigate to your project directory
cd c:\Users\sinadvd\Documents\VScode\homelab-prj\k8s-homelab\homelab

# Create the cluster using your configuration
kind create cluster --config kind-cluster.yaml

# Verify cluster is running
kubectl cluster-info
kubectl get nodes
```

**Expected Output:**
```
NAME                    STATUS   ROLES           AGE   VERSION
homelab-control-plane   Ready    control-plane   1m    v1.27.3
homelab-worker          Ready    <none>          1m    v1.27.3
homelab-worker2         Ready    <none>          1m    v1.27.3
```

### Step 2: Basic kubectl Commands
```powershell
# Check cluster status
kubectl get all --all-namespaces

# View cluster information
kubectl cluster-info
kubectl version

# List available API resources
kubectl api-resources
```

---

## Phase 2: Your First Pod and Deployment (Nginx)

### Step 3: Create Your First Pod

**Create:** `01-nginx-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: learning
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"
```

**Apply and Test:**
```powershell
# Apply the pod
kubectl apply -f 01-nginx-pod.yaml

# Check pod status
kubectl get pods
kubectl describe pod nginx-pod

# Check pod logs
kubectl logs nginx-pod

# Access the pod (port forward)
kubectl port-forward nginx-pod 8080:80
# Open browser to http://localhost:8080
```

**Learning Objectives:**
- ✅ Understand Pod structure
- ✅ Learn resource requests/limits
- ✅ Practice kubectl commands
- ✅ Understand port forwarding

### Step 4: Scale with Deployments

**Create:** `02-nginx-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Apply and Experiment:**
```powershell
# Delete the single pod first
kubectl delete pod nginx-pod

# Apply the deployment
kubectl apply -f 02-nginx-deployment.yaml

# Watch pods being created
kubectl get pods -w

# Check deployment status
kubectl get deployments
kubectl describe deployment nginx-deployment

# Scale the deployment
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods

# Scale back down
kubectl scale deployment nginx-deployment --replicas=2
```

**Learning Objectives:**
- ✅ Understand Deployments vs Pods
- ✅ Learn about ReplicaSets
- ✅ Practice scaling applications
- ✅ Understand health checks (probes)

---

## Phase 3: Networking and Services

### Step 5: Expose Your Application

**Create:** `03-nginx-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30002  # This matches your kind-cluster.yaml port mapping
  type: NodePort
```

**Apply and Test:**
```powershell
# Apply the service
kubectl apply -f 03-nginx-service.yaml

# Check service
kubectl get services
kubectl describe service nginx-service

# Test the service internally
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- nginx-service

# Test from your host machine (thanks to KIND port mapping)
# Open browser to http://localhost:30002
```

**Learning Objectives:**
- ✅ Understand Services and service discovery
- ✅ Learn about NodePort, ClusterIP service types
- ✅ Practice internal networking

### Step 6: Advanced Service Discovery

**Create:** `04-multiple-services.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  clusterIP: None
```

**Test Different Service Types:**
```powershell
kubectl apply -f 04-multiple-services.yaml

# Test service discovery
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup nginx-service
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup nginx-clusterip
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup nginx-headless
```

---

## Phase 4: Configuration Management

### Step 7: ConfigMaps for Configuration

**Create:** `05-nginx-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 80;
            location / {
                root /usr/share/nginx/html;
                index index.html;
            }
            location /health {
                return 200 "OK\n";
                add_header Content-Type text/plain;
            }
        }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Learning Kubernetes!</title>
    </head>
    <body>
        <h1>Hello from Kubernetes!</h1>
        <p>This is served from a ConfigMap</p>
        <p>Pod: ${HOSTNAME}</p>
    </body>
    </html>
```

**Create:** `06-nginx-with-config.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-config
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-config
  template:
    metadata:
      labels:
        app: nginx-config
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: html-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
      - name: html-volume
        configMap:
          name: nginx-config
```

**Apply and Test:**
```powershell
kubectl apply -f 05-nginx-configmap.yaml
kubectl apply -f 06-nginx-with-config.yaml

# Create service for the new deployment
kubectl expose deployment nginx-with-config --port=80 --type=NodePort

# Test the custom configuration
kubectl get services
# Access via the NodePort assigned
```

**Learning Objectives:**
- ✅ Understand ConfigMaps
- ✅ Learn volume mounting
- ✅ Practice environment variables
- ✅ Understand configuration separation

---

## Phase 5: Debugging and Troubleshooting (Busybox)

### Step 8: Debugging Tools

**Create:** `07-debug-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ['sleep', '3600']
    resources:
      requests:
        memory: "32Mi"
        cpu: "10m"
```

**Debugging Commands:**
```powershell
kubectl apply -f 07-debug-pod.yaml

# Execute commands in the pod
kubectl exec -it debug-pod -- /bin/sh

# Inside the pod, try these commands:
# wget -qO- nginx-service
# nslookup nginx-service
# ping nginx-service
# env | grep KUBERNETES

# From outside, check pod details
kubectl describe pod debug-pod
kubectl logs debug-pod

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Step 9: Troubleshooting Common Issues

**Create:** `08-broken-pod.yaml` (Intentionally broken for learning)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
spec:
  containers:
  - name: broken-container
    image: nginx:nonexistent-tag
    ports:
    - containerPort: 80
```

**Troubleshooting Practice:**
```powershell
kubectl apply -f 08-broken-pod.yaml

# Observe the failure
kubectl get pods
kubectl describe pod broken-pod
kubectl logs broken-pod

# Fix the issue
kubectl delete pod broken-pod
# Edit the file to use nginx:latest
kubectl apply -f 08-broken-pod.yaml
```

**Learning Objectives:**
- ✅ Master kubectl exec for debugging
- ✅ Learn log analysis
- ✅ Practice troubleshooting workflows
- ✅ Understand common failure patterns

---

## Phase 6: Persistent Storage (Redis)

### Step 10: Stateless vs Stateful Applications

**Create:** `09-redis-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

**Test Data Persistence:**
```powershell
kubectl apply -f 09-redis-deployment.yaml

# Expose Redis
kubectl expose deployment redis-deployment --port=6379 --type=ClusterIP

# Connect to Redis and add data
kubectl exec -it deployment/redis-deployment -- redis-cli
# Inside Redis CLI:
# SET mykey "Hello Kubernetes"
# GET mykey
# exit

# Delete the pod and see data loss
kubectl delete pod -l app=redis
kubectl get pods
# Wait for new pod to start, then check data
kubectl exec -it deployment/redis-deployment -- redis-cli GET mykey
```

### Step 11: Persistent Volumes

**Create:** `10-redis-persistent.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-statefulset
spec:
  serviceName: redis-headless
  replicas: 1
  selector:
    matchLabels:
      app: redis-persistent
  template:
    metadata:
      labels:
        app: redis-persistent
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-storage
          mountPath: /data
        command: ["redis-server", "--appendonly", "yes"]
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-persistent-service
spec:
  selector:
    app: redis-persistent
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

**Test Persistent Storage:**
```powershell
# Clean up previous Redis
kubectl delete deployment redis-deployment
kubectl delete service redis-deployment

# Apply persistent version
kubectl apply -f 10-redis-persistent.yaml

# Wait for StatefulSet to be ready
kubectl get statefulsets
kubectl get pvc

# Add data
kubectl exec -it redis-statefulset-0 -- redis-cli
# SET persistent-key "This will survive pod restarts"
# GET persistent-key
# exit

# Delete the pod
kubectl delete pod redis-statefulset-0

# Wait for automatic recreation and check data
kubectl get pods
kubectl exec -it redis-statefulset-0 -- redis-cli GET persistent-key
```

**Learning Objectives:**
- ✅ Understand StatefulSets vs Deployments
- ✅ Learn Persistent Volumes and Claims
- ✅ Practice data persistence
- ✅ Understand stateful application patterns

---

## Phase 7: Package Management with Helm

### Step 13: Introduction to Helm

Helm is the package manager for Kubernetes, often called "the apt/yum for Kubernetes." It helps you manage Kubernetes applications through packages called charts.

**Install Helm:**
```powershell
# Install Helm using winget
winget install Helm.Helm

# Verify installation
helm version

# Add the official stable repository
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Step 14: Your First Helm Chart

**Create Your First Chart:**
```powershell
# Create a new chart
helm create my-nginx-chart

# Navigate to the chart directory
cd my-nginx-chart

# Examine the chart structure
ls
```

**Customize the Chart:**

**Edit:** `my-nginx-chart/values.yaml`
```yaml
# Default values for my-nginx-chart
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: NodePort
  port: 80
  nodePort: 30003

ingress:
  enabled: false

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

autoscaling:
  enabled: false

nodeSelector: {}
tolerations: []
affinity: {}
```

**Edit:** `my-nginx-chart/templates/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-nginx-chart.fullname" . }}
  labels:
    {{- include "my-nginx-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-nginx-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-nginx-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**Edit:** `my-nginx-chart/templates/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-nginx-chart.fullname" . }}
  labels:
    {{- include "my-nginx-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if eq .Values.service.type "NodePort" }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    {{- include "my-nginx-chart.selectorLabels" . | nindent 4 }}
```

### Step 15: Deploy with Helm

**Deploy Your Chart:**
```powershell
# Navigate back to the main directory
cd ..

# Install the chart
helm install my-nginx ./my-nginx-chart

# Check the release
helm list
kubectl get all -l app.kubernetes.io/instance=my-nginx

# Test the deployment
# Open browser to http://localhost:30003
```

**Manage Helm Releases:**
```powershell
# Upgrade the release with new values
helm upgrade my-nginx ./my-nginx-chart --set replicaCount=3

# Check the upgrade
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# View release history
helm history my-nginx

# Rollback if needed
helm rollback my-nginx 1

# Uninstall the release
helm uninstall my-nginx
```

### Step 16: Using Public Helm Charts

**Deploy Redis using Bitnami Chart:**
```powershell
# Search for Redis charts
helm search repo redis

# Install Redis using Bitnami chart
helm install my-redis bitnami/redis \
  --set auth.enabled=false \
  --set master.persistence.enabled=false \
  --set replica.persistence.enabled=false

# Check the installation
kubectl get all -l app.kubernetes.io/instance=my-redis

# Get Redis connection info
helm status my-redis

# Test Redis connection
kubectl run redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:7.0-debian-11 -- bash
# Inside the pod:
# redis-cli -h my-redis-master
# set test-key "Hello from Helm Redis"
# get test-key
# exit
```

**Deploy WordPress with MySQL:**
```powershell
# Install WordPress with MySQL
helm install my-wordpress bitnami/wordpress \
  --set service.type=NodePort \
  --set service.nodePorts.http=30004 \
  --set wordpressUsername=admin \
  --set wordpressPassword=password123 \
  --set mariadb.primary.persistence.enabled=false

# Wait for deployment
kubectl get pods -l app.kubernetes.io/instance=my-wordpress -w

# Get WordPress credentials
echo "WordPress URL: http://localhost:30004"
echo "Username: admin"
echo "Password: password123"

# Clean up
helm uninstall my-wordpress
helm uninstall my-redis
```

**Learning Objectives:**
- ✅ Understand Helm charts and templating
- ✅ Learn to create custom charts
- ✅ Practice using public chart repositories
- ✅ Master Helm release management

---

## Phase 8: GitOps with ArgoCD

### Step 17: Install ArgoCD

**Deploy ArgoCD:**

**Create:** `argocd-install.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-server
  namespace: argocd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
  namespace: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: argocd-server
  template:
    metadata:
      labels:
        app: argocd-server
    spec:
      serviceAccountName: argocd-server
      containers:
      - name: argocd-server
        image: quay.io/argoproj/argocd:v2.8.4
        ports:
        - containerPort: 8080
        - containerPort: 8083
        command:
        - argocd-server
        - --insecure
        - --staticassets
        - /shared/app
        env:
        - name: ARGOCD_SERVER_INSECURE
          value: "true"
        volumeMounts:
        - name: static-files
          mountPath: /shared
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: static-files
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
spec:
  selector:
    app: argocd-server
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30005
  - name: grpc
    port: 443
    targetPort: 8080
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: argocd-repo-server
  template:
    metadata:
      labels:
        app: argocd-repo-server
    spec:
      containers:
      - name: argocd-repo-server
        image: quay.io/argoproj/argocd:v2.8.4
        ports:
        - containerPort: 8081
        command:
        - argocd-repo-server
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  selector:
    app: argocd-repo-server
  ports:
  - port: 8081
    targetPort: 8081
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-application-controller
  namespace: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: argocd-application-controller
  template:
    metadata:
      labels:
        app: argocd-application-controller
    spec:
      containers:
      - name: argocd-application-controller
        image: quay.io/argoproj/argocd:v2.8.4
        command:
        - argocd-application-controller
        - --status-processors
        - "20"
        - --operation-processors
        - "10"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
# Create admin user secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-initial-admin-secret
  namespace: argocd
type: Opaque
data:
  password: YWRtaW4xMjM=  # admin123 (base64 encoded)
```

**Install ArgoCD:**
```powershell
# Apply ArgoCD installation
kubectl apply -f argocd-install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Access ArgoCD UI
echo "ArgoCD URL: http://localhost:30005"
echo "Username: admin"
echo "Password: admin123"
```

### Step 18: Prepare Git Repository Structure

**Create Application Manifests:**

**Create:** `gitops-apps/nginx-app/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-nginx
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gitops-nginx
  template:
    metadata:
      labels:
        app: gitops-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: gitops-nginx-service
  namespace: default
spec:
  selector:
    app: gitops-nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30006
  type: NodePort
```

**Create:** `gitops-apps/redis-app/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-redis
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitops-redis
  template:
    metadata:
      labels:
        app: gitops-redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: gitops-redis-service
  namespace: default
spec:
  selector:
    app: gitops-redis
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

### Step 19: Create ArgoCD Applications

**Create:** `argocd-nginx-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'file:///c/Users/sinadvd/Documents/VScode/homelab-prj/k8s-homelab/homelab'
    targetRevision: HEAD
    path: gitops-apps/nginx-app
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Create:** `argocd-redis-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'file:///c/Users/sinadvd/Documents/VScode/homelab-prj/k8s-homelab/homelab'
    targetRevision: HEAD
    path: gitops-apps/redis-app
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Step 20: Deploy Applications with ArgoCD

**Deploy Applications:**
```powershell
# Create the gitops-apps directory structure
mkdir -p gitops-apps/nginx-app
mkdir -p gitops-apps/redis-app

# Create the application manifests (copy the YAML content above)

# Deploy ArgoCD applications
kubectl apply -f argocd-nginx-app.yaml
kubectl apply -f argocd-redis-app.yaml

# Check ArgoCD applications
kubectl get applications -n argocd

# Watch the applications deploy
kubectl get pods -w
```

**Access ArgoCD UI:**
1. Open browser to http://localhost:30005
2. Login with admin/admin123
3. You should see both nginx-app and redis-app applications
4. Click on each application to see the deployment details

### Step 21: GitOps Workflow Demonstration

**Simulate GitOps Changes:**
```powershell
# Update the nginx deployment to use a different image
# Edit gitops-apps/nginx-app/deployment.yaml
# Change image from nginx:1.21 to nginx:1.22

# In the ArgoCD UI, you'll see the application go "OutOfSync"
# Click "Sync" to apply the changes, or wait for auto-sync

# Scale the application
# Change replicas from 2 to 4 in the deployment.yaml

# Again, ArgoCD will detect the drift and sync automatically
```

**Test Application Updates:**
```powershell
# Create a ConfigMap for nginx
cat > gitops-apps/nginx-app/configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>GitOps with ArgoCD</title></head>
    <body>
        <h1>Hello from GitOps!</h1>
        <p>This application is managed by ArgoCD</p>
        <p>Version: 2.0</p>
    </body>
    </html>
EOF

# Update the deployment to use the ConfigMap
# Add volume and volumeMount to the deployment.yaml
```

**Updated deployment with ConfigMap:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-nginx
  namespace: default
spec:
  replicas: 4  # Updated from 2
  selector:
    matchLabels:
      app: gitops-nginx
  template:
    metadata:
      labels:
        app: gitops-nginx
    spec:
      containers:
      - name: nginx
        image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        - name: config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: config-volume
        configMap:
          name: nginx-config
```

### Step 22: App of Apps Pattern

**Create:** `argocd-app-of-apps.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'file:///c/Users/sinadvd/Documents/VScode/homelab-prj/k8s-homelab/homelab'
    targetRevision: HEAD
    path: gitops-apps
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Create Application Definitions:**

**Create:** `gitops-apps/nginx-application.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app-managed
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'file:///c/Users/sinadvd/Documents/VScode/homelab-prj/k8s-homelab/homelab'
    targetRevision: HEAD
    path: gitops-apps/nginx-app
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Deploy App of Apps:**
```powershell
# Deploy the app of apps
kubectl apply -f argocd-app-of-apps.yaml

# Check all applications
kubectl get applications -n argocd

# Test the nginx application
kubectl get pods -n production
# Open browser to http://localhost:30006
```

**Learning Objectives:**
- ✅ Understand GitOps principles and workflow
- ✅ Deploy and configure ArgoCD
- ✅ Create and manage ArgoCD applications
- ✅ Practice declarative application management
- ✅ Learn the App of Apps pattern

---

## Phase 9: Complete Application Stack with Helm and ArgoCD

### Step 23: Create a Helm Chart for Complete Stack

**Create Complete Application Helm Chart:**
```powershell
# Create a new chart for our complete stack
helm create complete-stack-chart
cd complete-stack-chart
```

**Edit:** `complete-stack-chart/values.yaml`
```yaml
# Complete stack values
frontend:
  replicaCount: 2
  image:
    repository: nginx
    tag: "latest"
  service:
    type: NodePort
    port: 80
    nodePort: 30007

backend:
  replicaCount: 2
  image:
    repository: httpd
    tag: "latest"
  service:
    port: 8080

redis:
  enabled: true
  image:
    repository: redis
    tag: "alpine"
  persistence: false

postgres:
  enabled: true
  image:
    repository: postgres
    tag: "15"
  database: "appdb"
  username: "appuser"
  password: "apppass123"
  persistence: false

ingress:
  enabled: true
  host: "complete-app.local"

configMap:
  frontendHtml: |
    <!DOCTYPE html>
    <html>
    <head><title>Complete Stack</title></head>
    <body>
        <h1>Complete Application Stack</h1>
        <p>Frontend: Nginx</p>
        <p>Backend: Apache HTTP Server</p>
        <p>Cache: Redis</p>
        <p>Database: PostgreSQL</p>
        <p>Deployed with: Helm + ArgoCD</p>
    </body>
    </html>
```

**Create:** `complete-stack-chart/templates/frontend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "complete-stack-chart.fullname" . }}-frontend
  labels:
    {{- include "complete-stack-chart.labels" . | nindent 4 }}
    component: frontend
spec:
  replicas: {{ .Values.frontend.replicaCount }}
  selector:
    matchLabels:
      {{- include "complete-stack-chart.selectorLabels" . | nindent 6 }}
      component: frontend
  template:
    metadata:
      labels:
        {{- include "complete-stack-chart.selectorLabels" . | nindent 8 }}
        component: frontend
    spec:
      containers:
        - name: frontend
          image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          volumeMounts:
            - name: frontend-config
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: frontend-config
          configMap:
            name: frontend-config
```

**Create:** `complete-stack-chart/templates/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "complete-stack-chart.fullname" . }}-config
  labels:
    {{- include "complete-stack-chart.labels" . | nindent 4 }}
data:
  index.html: {{ .Values.configMap.frontendHtml | quote }}
```

**Create:** `complete-stack-chart/templates/redis-deployment.yaml`
```yaml
{{- if .Values.redis.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "complete-stack-chart.fullname" . }}-redis
  labels:
    {{- include "complete-stack-chart.labels" . | nindent 4 }}
    component: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      {{- include "complete-stack-chart.selectorLabels" . | nindent 6 }}
      component: redis
  template:
    metadata:
      labels:
        {{- include "complete-stack-chart.selectorLabels" . | nindent 8 }}
        component: redis
    spec:
      containers:
        - name: redis
          image: "{{ .Values.redis.image.repository }}:{{ .Values.redis.image.tag }}"
          ports:
            - name: redis
              containerPort: 6379
              protocol: TCP
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
{{- end }}
```

### Step 24: Deploy with ArgoCD and Helm

**Create ArgoCD Application for Helm Chart:**

**Create:** `argocd-helm-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: complete-stack-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'file:///c/Users/sinadvd/Documents/VScode/homelab-prj/k8s-homelab/homelab'
    targetRevision: HEAD
    path: complete-stack-chart
    helm:
      valueFiles:
      - values.yaml
      parameters:
      - name: frontend.replicaCount
        value: "3"
      - name: ingress.host
        value: "my-complete-app.local"
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: helm-apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Deploy the Helm Application:**
```powershell
# Go back to main directory
cd ..

# Deploy the ArgoCD application that manages the Helm chart
kubectl apply -f argocd-helm-app.yaml

# Watch the deployment
kubectl get pods -n helm-apps -w

# Check the ArgoCD UI to see the Helm application
```

**Test the Complete Stack:**
```powershell
# Check all resources
kubectl get all -n helm-apps

# Add host entry
echo "127.0.0.1 my-complete-app.local" >> C:\Windows\System32\drivers\etc\hosts

# Test the application
# Browser: http://my-complete-app.local:30080 (if ingress is working)
# Or check the NodePort service
kubectl get services -n helm-apps
```

**Learning Objectives:**
- ✅ Combine Helm charts with ArgoCD
- ✅ Manage complex multi-component applications
- ✅ Use Helm parameters in ArgoCD
- ✅ Practice GitOps with templated applications

---

## Phase 10: Ingress and External Traffic Management

### Step 25: Install and Configure Ingress Controller

**Create:** `25-ingress-nginx-controller.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress-controller
  template:
    metadata:
      labels:
        app: nginx-ingress-controller
    spec:
      containers:
      - name: nginx-ingress-controller
        image: k8s.gcr.io/ingress-nginx/controller:v1.8.1
        ports:
        - containerPort: 80
        - containerPort: 443
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    name: http
  - port: 443
    targetPort: 443
    nodePort: 30443
    name: https
  selector:
    app: nginx-ingress-controller
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-services
  namespace: ingress-nginx
```

### Step 26: Create Multi-Service Application with Ingress

**Create:** `26-multi-app-stack.yaml`
```yaml
# Frontend Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: frontend-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      volumes:
      - name: frontend-config
        configMap:
          name: frontend-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Frontend App</title></head>
    <body>
        <h1>Frontend Application</h1>
        <p>This is the main application frontend</p>
        <a href="/api/health">Check API Health</a>
        <br><br>
        <a href="/api/users">View Users API</a>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
# API Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: httpd:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: api-config
          mountPath: /usr/local/apache2/htdocs/
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      volumes:
      - name: api-config
        configMap:
          name: api-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  health: |
    {
      "status": "healthy",
      "timestamp": "2025-01-01T00:00:00Z",
      "version": "1.0.0",
      "service": "api"
    }
  users: |
    {
      "users": [
        {"id": 1, "name": "Alice", "role": "admin"},
        {"id": 2, "name": "Bob", "role": "user"},
        {"id": 3, "name": "Charlie", "role": "developer"}
      ],
      "total": 3
    }
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
```

**Create:** `27-ingress-rules.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: homelab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: api.homelab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
---
# Additional ingress with SSL termination (demo)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.homelab.local
    secretName: tls-secret
  rules:
  - host: secure.homelab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Deploy and Test:**
```powershell
# Deploy ingress controller
kubectl apply -f 25-ingress-nginx-controller.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app=nginx-ingress-controller --timeout=300s

# Deploy applications
kubectl apply -f 26-multi-app-stack.yaml
kubectl apply -f 27-ingress-rules.yaml

# Add host entries (run as Administrator)
echo "127.0.0.1 homelab.local" >> C:\Windows\System32\drivers\etc\hosts
echo "127.0.0.1 api.homelab.local" >> C:\Windows\System32\drivers\etc\hosts
echo "127.0.0.1 secure.homelab.local" >> C:\Windows\System32\drivers\etc\hosts

# Test ingress
# Browser: http://homelab.local:30080
# Browser: http://homelab.local:30080/api/health
# Browser: http://api.homelab.local:30080/users
```

**Learning Objectives:**
- ✅ Understand Ingress controllers and rules
- ✅ Learn path-based and host-based routing
- ✅ Practice SSL termination and redirects
- ✅ Understand ingress annotations

---

## Phase 11: Secrets Management and Security

### Step 27: Working with Secrets

**Create:** `28-secrets-demo.yaml`
```yaml
# Database credentials secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=  # admin (base64 encoded)
  password: UGFzc3dvcmQxMjM=  # Password123 (base64 encoded)
  host: cG9zdGdyZXMtc2VydmljZQ==  # postgres-service (base64)
---
# API Keys secret
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
type: Opaque
stringData:  # Using stringData for easier management
  stripe-key: "sk_test_123456789"
  github-token: "ghp_abcdefghijklmnopqrstuvwxyz"
  jwt-secret: "my-super-secret-jwt-key"
---
# TLS Secret for HTTPS
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # Your certificate (base64 encoded)
  tls.key: LS0tLS1CRUdJTi... # Your private key (base64 encoded)
---
# PostgreSQL with secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: POSTGRES_DB
          value: homelab
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: init-scripts
          mountPath: /docker-entrypoint-initdb.d
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-storage
        emptyDir: {}
      - name: init-scripts
        configMap:
          name: postgres-init
---
# PostgreSQL initialization scripts
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
data:
  01-init.sql: |
    CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        username VARCHAR(50) UNIQUE NOT NULL,
        email VARCHAR(100) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    INSERT INTO users (username, email) VALUES
    ('admin', 'admin@homelab.local'),
    ('developer', 'dev@homelab.local'),
    ('user1', 'user1@homelab.local');
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# Application using secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
        env:
        - name: DATABASE_URL
          value: "postgresql://$(DB_USER):$(DB_PASS)@$(DB_HOST):5432/homelab"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: host
        - name: STRIPE_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: stripe-key
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: jwt-secret
        volumeMounts:
        - name: app-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      volumes:
      - name: app-config
        configMap:
          name: secure-app-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: secure-app-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Secure Application</title></head>
    <body>
        <h1>Secure Application</h1>
        <p>This application uses secrets for sensitive data</p>
        <ul>
            <li>Database credentials from secrets</li>
            <li>API keys from secrets</li>
            <li>JWT tokens from secrets</li>
        </ul>
        <p>Check the pod environment variables to see how secrets are injected</p>
    </body>
    </html>
```

**Create and Manage Secrets:**
```powershell
# Create secret from command line
kubectl create secret generic app-config `
    --from-literal=api-key="super-secret-api-key" `
    --from-literal=db-url="postgresql://user:pass@postgres:5432/db" `
    --from-literal=redis-password="redis-secret"

# Create secret from file
echo "my-secret-token-content" > token.txt
kubectl create secret generic file-secret --from-file=token.txt
rm token.txt

# Create TLS secret (self-signed for demo)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 `
    -keyout tls.key -out tls.crt `
    -subj "/CN=secure.homelab.local"
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
rm tls.crt, tls.key

# Apply the PostgreSQL with secrets
kubectl apply -f 28-secrets-demo.yaml

# Verify secrets (note: data is base64 encoded)
kubectl get secrets
kubectl describe secret db-credentials

# Test database connection
kubectl exec -it deployment/postgres -- psql -U admin -d homelab
# Inside PostgreSQL:
# \l
# \dt
# SELECT * FROM users;
# \q

# Check environment variables in secure app
kubectl exec -it deployment/secure-app -- env | grep -E "(DATABASE|STRIPE|JWT)"
```

### Step 28: RBAC (Role-Based Access Control)

**Create:** `29-rbac-demo.yaml`
```yaml
# Create namespaces for RBAC demo
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    team: alpha
    environment: development
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-beta
  labels:
    team: beta
    environment: production
---
# ServiceAccounts for different teams
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha-developer
  namespace: team-alpha
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha-admin
  namespace: team-alpha
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-beta-operator
  namespace: team-beta
---
# Role for developers (limited permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-alpha
  name: developer-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "watch", "list", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]
  verbs: ["get", "create"]
---
# Role for admins (full permissions in namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-alpha
  name: admin-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
# Role for production operators (read-only + restart)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-beta
  name: operator-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "patch"]  # patch for restart
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
# RoleBindings to connect ServiceAccounts to Roles
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: team-alpha
subjects:
- kind: ServiceAccount
  name: team-alpha-developer
  namespace: team-alpha
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-binding
  namespace: team-alpha
subjects:
- kind: ServiceAccount
  name: team-alpha-admin
  namespace: team-alpha
roleRef:
  kind: Role
  name: admin-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: operator-binding
  namespace: team-beta
subjects:
- kind: ServiceAccount
  name: team-beta-operator
  namespace: team-beta
roleRef:
  kind: Role
  name: operator-role
  apiGroup: rbac.authorization.k8s.io
---
# ClusterRole for cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "namespaces"]
  verbs: ["get", "list"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
---
# ClusterRoleBinding for cluster-wide access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: team-leads-cluster-access
subjects:
- kind: ServiceAccount
  name: team-alpha-admin
  namespace: team-alpha
- kind: ServiceAccount
  name: team-beta-operator
  namespace: team-beta
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
---
# Test Pods using different ServiceAccounts
apiVersion: v1
kind: Pod
metadata:
  name: developer-test-pod
  namespace: team-alpha
spec:
  serviceAccountName: team-alpha-developer
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep', '3600']
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
---
apiVersion: v1
kind: Pod
metadata:
  name: admin-test-pod
  namespace: team-alpha
spec:
  serviceAccountName: team-alpha-admin
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep', '3600']
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
---
apiVersion: v1
kind: Pod
metadata:
  name: operator-test-pod
  namespace: team-beta
spec:
  serviceAccountName: team-beta-operator
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep', '3600']
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
```

**Test RBAC:**
```powershell
# Apply RBAC configuration
kubectl apply -f 29-rbac-demo.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod --all --all-namespaces --timeout=300s

# Test developer permissions (should work)
kubectl exec -n team-alpha -it developer-test-pod -- kubectl get pods -n team-alpha
kubectl exec -n team-alpha -it developer-test-pod -- kubectl get services -n team-alpha

# Test developer permissions (should fail - no access to secrets)
kubectl exec -n team-alpha -it developer-test-pod -- kubectl get secrets -n team-alpha

# Test admin permissions (should work)
kubectl exec -n team-alpha -it admin-test-pod -- kubectl get secrets -n team-alpha
kubectl exec -n team-alpha -it admin-test-pod -- kubectl get nodes

# Test operator permissions (read-only access)
kubectl exec -n team-beta -it operator-test-pod -- kubectl get pods -n team-beta
kubectl exec -n team-beta -it operator-test-pod -- kubectl get nodes

# Test cross-namespace access (should fail)
kubectl exec -n team-alpha -it developer-test-pod -- kubectl get pods -n team-beta

# Check what each ServiceAccount can do
kubectl auth can-i get pods --as=system:serviceaccount:team-alpha:team-alpha-developer -n team-alpha
kubectl auth can-i delete pods --as=system:serviceaccount:team-alpha:team-alpha-developer -n team-alpha
kubectl auth can-i get secrets --as=system:serviceaccount:team-alpha:team-alpha-developer -n team-alpha
kubectl auth can-i get nodes --as=system:serviceaccount:team-alpha:team-alpha-admin
```

**Learning Objectives:**
- ✅ Understand Secrets vs ConfigMaps
- ✅ Learn secret types and best practices
- ✅ Master RBAC concepts (Roles, RoleBindings, ClusterRoles)
- ✅ Practice security principles
- ✅ Implement least privilege access

---

## Phase 12: Monitoring and Observability Stack

### Step 29: Deploy Prometheus

**Create:** `30-prometheus.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
# Prometheus ServiceAccount and RBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
---
# Prometheus ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
    - "/etc/prometheus/rules/*.yml"
    
    alerting:
      alertmanagers:
      - static_configs:
        - targets: []
    
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
    
    - job_name: 'kubernetes-services'
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_service_name
    
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics
    
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
  
  alerts.yml: |
    groups:
    - name: kubernetes
      rules:
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is restarting frequently"
      
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Container {{ $labels.container }} in pod {{ $labels.pod }} is using > 80% memory"
---
# Prometheus Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.47.0
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus/
        - name: storage-volume
          mountPath: /prometheus/
        args:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus/'
        - '--web.console.libraries=/etc/prometheus/console_libraries'
        - '--web.console.templates=/etc/prometheus/consoles'
        - '--storage.tsdb.retention.time=200h'
        - '--web.enable-lifecycle'
        - '--web.enable-admin-api'
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: storage-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30090
  type: NodePort
```

### Step 30: Deploy Grafana

**Create:** `31-grafana.yaml`
```yaml
# Grafana ConfigMap for datasources
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-service:9090
      access: proxy
      isDefault: true
      editable: true
---
# Grafana ConfigMap for dashboards
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  kubernetes-dashboard.json: |
    {
      "dashboard": {
        "id": null,
        "title": "Kubernetes Cluster Overview",
        "tags": ["kubernetes"],
        "timezone": "browser",
        "panels": [
          {
            "id": 1,
            "title": "Pod Status",
            "type": "stat",
            "targets": [
              {
                "expr": "kube_pod_status_phase",
                "legendFormat": "{{phase}}"
              }
            ],
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
          },
          {
            "id": 2,
            "title": "CPU Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(container_cpu_usage_seconds_total[5m])",
                "legendFormat": "{{pod}}"
              }
            ],
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
          }
        ],
        "time": {"from": "now-1h", "to": "now"},
        "refresh": "5s"
      }
    }
---
# Grafana Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:10.1.0
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin123"
        - name: GF_USERS_ALLOW_SIGN_UP
          value: "false"
        - name: GF_INSTALL_PLUGINS
          value: "grafana-kubernetes-app"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: datasources
          mountPath: /etc/grafana/provisioning/datasources
        - name: dashboards-config
          mountPath: /etc/grafana/provisioning/dashboards
        - name: dashboards
          mountPath: /var/lib/grafana/dashboards
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: grafana-storage
        emptyDir: {}
      - name: datasources
        configMap:
          name: grafana-datasources
      - name: dashboards-config
        configMap:
          name: grafana-dashboards-config
      - name: dashboards
        configMap:
          name: grafana-dashboards
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-config
  namespace: monitoring
data:
  dashboards.yaml: |
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      updateIntervalSeconds: 10
      options:
        path: /var/lib/grafana/dashboards
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300
  type: NodePort
```

### Step 31: Application with Metrics

**Create:** `32-app-with-metrics.yaml`
```yaml
# Sample application that exposes metrics
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: metrics-app
  template:
    metadata:
      labels:
        app: metrics-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
          name: http
        - containerPort: 8080
          name: metrics
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/
        - name: metrics-content
          mountPath: /usr/share/nginx/html/metrics
          subPath: metrics
        - name: app-content
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-metrics-config
      - name: metrics-content
        configMap:
          name: metrics-content
      - name: app-content
        configMap:
          name: app-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-metrics-config
data:
  default.conf: |
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /health {
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
    }
    server {
        listen 8080;
        location /metrics {
            root /usr/share/nginx/html;
            add_header Content-Type text/plain;
        }
        location /health {
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metrics-content
data:
  metrics: |
    # HELP nginx_requests_total Total number of nginx requests
    # TYPE nginx_requests_total counter
    nginx_requests_total{method="GET",status="200",pod="metrics-app"} 1542
    nginx_requests_total{method="POST",status="200",pod="metrics-app"} 234
    nginx_requests_total{method="GET",status="404",pod="metrics-app"} 45
    nginx_requests_total{method="GET",status="500",pod="metrics-app"} 12
    
    # HELP nginx_request_duration_seconds Request duration in seconds
    # TYPE nginx_request_duration_seconds histogram
    nginx_request_duration_seconds_bucket{le="0.1"} 1200
    nginx_request_duration_seconds_bucket{le="0.5"} 1450
    nginx_request_duration_seconds_bucket{le="1.0"} 1550
    nginx_request_duration_seconds_bucket{le="2.0"} 1580
    nginx_request_duration_seconds_bucket{le="+Inf"} 1583
    nginx_request_duration_seconds_sum 425.3
    nginx_request_duration_seconds_count 1583
    
    # HELP nginx_memory_usage_bytes Memory usage in bytes
    # TYPE nginx_memory_usage_bytes gauge
    nginx_memory_usage_bytes 67108864
    
    # HELP nginx_connections_active Currently active connections
    # TYPE nginx_connections_active gauge
    nginx_connections_active 15
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Metrics Application</title></head>
    <body>
        <h1>Metrics Demo Application</h1>
        <p>This application exposes Prometheus metrics</p>
        <ul>
            <li><a href="/metrics">View Metrics</a></li>
            <li><a href="/health">Health Check</a></li>
        </ul>
        <h2>Metrics Available:</h2>
        <ul>
            <li>nginx_requests_total - Total HTTP requests</li>
            <li>nginx_request_duration_seconds - Request duration histogram</li>
            <li>nginx_memory_usage_bytes - Memory usage</li>
            <li>nginx_connections_active - Active connections</li>
        </ul>
        <p>Pod: <span id="hostname">Loading...</span></p>
        <script>
            fetch('/hostname').then(r => r.text()).then(text => {
                document.getElementById('hostname').textContent = text || window.location.hostname;
            }).catch(() => {
                document.getElementById('hostname').textContent = window.location.hostname;
            });
        </script>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-app-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: metrics-app
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: metrics
    port: 8080
    targetPort: 8080
---
# Load generator to create metrics
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: load-generator
        image: busybox:latest
        command: ['sh', '-c']
        args:
        - |
          while true; do
            wget -qO- http://metrics-app-service/ > /dev/null
            wget -qO- http://metrics-app-service/health > /dev/null
            wget -qO- http://metrics-app-service/metrics > /dev/null
            sleep $((RANDOM % 5 + 1))
          done
        resources:
          requests:
            memory: "32Mi"
            cpu: "10m"
          limits:
            memory: "64Mi"
            cpu: "50m"
```

**Deploy and Configure Monitoring:**
```powershell
# Deploy monitoring stack
kubectl apply -f 30-prometheus.yaml
kubectl apply -f 31-grafana.yaml
kubectl apply -f 32-app-with-metrics.yaml

# Wait for pods to be ready
kubectl wait --namespace monitoring --for=condition=ready pod --selector=app=prometheus --timeout=300s
kubectl wait --namespace monitoring --for=condition=ready pod --selector=app=grafana --timeout=300s

# Access Prometheus: http://localhost:30090
# Access Grafana: http://localhost:30300 (admin/admin123)

# Test metrics endpoint
kubectl port-forward service/metrics-app-service 8080:8080
# Open: http://localhost:8080/metrics in another terminal
```

**Grafana Dashboard Setup:**
1. Login to Grafana (admin/admin123)
2. Go to Dashboards → Import
3. Create new dashboard with panels:
   - **Request Rate**: `rate(nginx_requests_total[5m])`
   - **Error Rate**: `rate(nginx_requests_total{status!~"2.."}[5m])`
   - **Response Time**: `histogram_quantile(0.95, nginx_request_duration_seconds_bucket)`
   - **Memory Usage**: `nginx_memory_usage_bytes`

**Learning Objectives:**
- ✅ Deploy and configure Prometheus
- ✅ Set up Grafana with data sources
- ✅ Understand metrics collection and scraping
- ✅ Create monitoring dashboards
- ✅ Practice alerting and visualization

---

## Phase 13: Advanced Workload Patterns

### Step 32: Init Containers and Sidecar Patterns

**Create:** `33-advanced-patterns.yaml`
```yaml
# Application with Init Container and Multiple Sidecars
apiVersion: apps/v1
kind: Deployment
metadata:
  name: advanced-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: advanced-app
  template:
    metadata:
      labels:
        app: advanced-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Init Containers - run sequentially before main containers
      initContainers:
      # 1. Database migration init container
      - name: db-migration
        image: postgres:15
        command: ['sh', '-c']
        args:
        - |
          echo "Starting database migration..."
          until pg_isready -h postgres-service -p 5432; do
            echo "Waiting for database..."
            sleep 5
          done
          echo "Database is ready!"
          
          # Run migrations (simulated)
          psql -h postgres-service -U admin -d homelab -c "
            CREATE TABLE IF NOT EXISTS migrations (
              id SERIAL PRIMARY KEY,
              version VARCHAR(50),
              applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
            INSERT INTO migrations (version) VALUES ('v1.2.0')
              ON CONFLICT DO NOTHING;
          "
          echo "Database migration completed!"
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      
      # 2. Configuration setup init container
      - name: config-setup
        image: busybox:latest
        command: ['sh', '-c']
        args:
        - |
          echo "Setting up application configuration..."
          echo '{"app": "advanced-app", "version": "1.2.0", "env": "production"}' > /shared/config.json
          echo "generating-app-key-$(date +%s)" > /shared/app-key.txt
          echo "Configuration setup complete!"
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      
      containers:
      # Main Application Container
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html/config
        - name: app-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
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
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      
      # Sidecar 1: Log processor
      - name: log-processor
        image: busybox:latest
        command: ['sh', '-c']
        args:
        - |
          echo "Log processor started..."
          while true; do
            echo "$(date): Processing application logs..."
            if [ -f /shared/config.json ]; then
              echo "$(date): App config: $(cat /shared/config.json)"
            fi
            echo "$(date): Logs processed and forwarded"
            sleep 30
          done
        volumeMounts:
        - name: shared-data
          mountPath: /shared
        - name: log-storage
          mountPath: /logs
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      
      # Sidecar 2: Metrics exporter
      - name: metrics-exporter
        image: nginx:alpine
        ports:
        - containerPort: 9090
          name: metrics
        volumeMounts:
        - name: metrics-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        - name: shared-data
          mountPath: /shared
        command: ['sh', '-c']
        args:
        - |
          # Generate dynamic metrics
          while true; do
            cat > /usr/share/nginx/html/metrics << EOF
          # HELP app_status Application status
          # TYPE app_status gauge
          app_status 1
          
          # HELP app_requests_total Total requests processed
          # TYPE app_requests_total counter
          app_requests_total $((RANDOM % 10000))
          
          # HELP app_memory_usage_bytes Memory usage in bytes
          # TYPE app_memory_usage_bytes gauge
          app_memory_usage_bytes $((RANDOM % 1000000000))
          
          # HELP app_cpu_usage_percent CPU usage percentage
          # TYPE app_cpu_usage_percent gauge
          app_cpu_usage_percent $((RANDOM % 100))
          EOF
            sleep 15
          done &
          nginx -g 'daemon off;'
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      
      # Sidecar 3: Configuration watcher
      - name: config-watcher
        image: busybox:latest
        command: ['sh', '-c']
        args:
        - |
          echo "Configuration watcher started..."
          last_config=""
          while true; do
            if [ -f /shared/config.json ]; then
              current_config=$(cat /shared/config.json)
              if [ "$current_config" != "$last_config" ]; then
                echo "$(date): Configuration changed!"
                echo "New config: $current_config"
                last_config="$current_config"
                # Notify main application (simulated)
                echo "$(date): Notified main application of config change"
              fi
            fi
            sleep 10
          done
        volumeMounts:
        - name: shared-data
          mountPath: /shared
        resources:
          requests:
            memory: "32Mi"
            cpu: "25m"
          limits:
            memory: "64Mi"
            cpu: "50m"
      
      volumes:
      - name: shared-data
        emptyDir: {}
      - name: log-storage
        emptyDir: {}
      - name: app-config
        configMap:
          name: advanced-app-config
      - name: nginx-config
        configMap:
          name: nginx-advanced-config
      - name: metrics-config
        configMap:
          name: metrics-nginx-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: advanced-app-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Advanced Application Pattern</title></head>
    <body>
        <h1>Advanced Application with Patterns</h1>
        <h2>Architecture:</h2>
        <ul>
            <li><strong>Init Containers:</strong>
                <ul>
                    <li>Database Migration - Sets up database schema</li>
                    <li>Configuration Setup - Prepares app configuration</li>
                </ul>
            </li>
            <li><strong>Main Container:</strong>
                <ul>
                    <li>Nginx Application Server</li>
                </ul>
            </li>
            <li><strong>Sidecar Containers:</strong>
                <ul>
                    <li>Log Processor - Handles log aggregation</li>
                    <li>Metrics Exporter - Exposes application metrics</li>
                    <li>Configuration Watcher - Monitors config changes</li>
                </ul>
            </li>
        </ul>
        <h2>Endpoints:</h2>
        <ul>
            <li><a href="/config/config.json">App Configuration</a></li>
            <li><a href="/health">Health Check</a></li>
            <li><a href="/ready">Readiness Check</a></li>
        </ul>
        <p><strong>Pod:</strong> {{ HOSTNAME }}</p>
    </body>
    </html>
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-advanced-config
data:
  default.conf: |
    server {
        listen 80;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /config/ {
            alias /usr/share/nginx/html/config/;
            add_header Content-Type application/json;
        }
        
        location /health {
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
        
        location /ready {
            return 200 "READY\n";
            add_header Content-Type text/plain;
        }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metrics-nginx-config
data:
  default.conf: |
    server {
        listen 9090;
        
        location /metrics {
            root /usr/share/nginx/html;
            add_header Content-Type text/plain;
        }
        
        location /health {
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: advanced-app-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: advanced-app
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30008
  - name: metrics
    port: 9090
    targetPort: 9090
  type: NodePort
```

### Step 33: Jobs and CronJobs

**Create:** `34-jobs-cronjobs.yaml`
```yaml
# One-time Job for data processing
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing-job
spec:
  template:
    spec:
      containers:
      - name: processor
        image: postgres:15
        command: ['sh', '-c']
        args:
        - |
          echo "Starting data processing job..."
          echo "Job started at: $(date)"
          
          # Connect to database and process data
          until pg_isready -h postgres-service -p 5432; do
            echo "Waiting for database..."
            sleep 5
          done
          
          echo "Processing user data..."
          psql -h postgres-service -U admin -d homelab -c "
            CREATE TABLE IF NOT EXISTS user_stats (
              id SERIAL PRIMARY KEY,
              total_users INTEGER,
              active_users INTEGER,
              processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
            
            INSERT INTO user_stats (total_users, active_users)
            SELECT COUNT(*), COUNT(*) * 0.8 FROM users;
          "
          
          echo "Processing completed at: $(date)"
          echo "Data processing job finished successfully!"
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      
      restartPolicy: Never
  backoffLimit: 3
  activeDeadlineSeconds: 300  # 5 minutes timeout
---
# Parallel Job for batch processing
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-batch-job
spec:
  parallelism: 4      # Run 4 pods in parallel
  completions: 12     # Need 12 successful completions
  template:
    spec:
      containers:
      - name: worker
        image: busybox:latest
        command: ['sh', '-c']
        args:
        - |
          worker_id="${HOSTNAME##*-}"
          echo "Worker $worker_id started at $(date)"
          
          # Simulate batch processing
          batch_size=$((RANDOM % 50 + 10))
          echo "Processing batch of $batch_size items..."
          
          for i in $(seq 1 $batch_size); do
            echo "Processing item $i/$batch_size"
            sleep 0.1
          done
          
          # Simulate some failures occasionally
          if [ $((RANDOM % 10)) -eq 0 ]; then
            echo "Worker $worker_id failed!"
            exit 1
          fi
          
          echo "Worker $worker_id completed successfully at $(date)"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      restartPolicy: Never
  backoffLimit: 6
---
# CronJob for regular database backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "*/3 * * * *"  # Every 3 minutes for demo (use "0 2 * * *" for daily at 2 AM)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command: ['sh', '-c']
            args:
            - |
              echo "Starting database backup at $(date)"
              backup_file="backup-$(date +%Y%m%d-%H%M%S).sql"
              
              # Wait for database
              until pg_isready -h postgres-service -p 5432; do
                echo "Waiting for database..."
                sleep 5
              done
              
              # Create backup
              echo "Creating backup: $backup_file"
              pg_dump -h postgres-service -U admin homelab > /backups/$backup_file
              
              # Verify backup
              if [ -f "/backups/$backup_file" ]; then
                backup_size=$(stat -c%s "/backups/$backup_file")
                echo "Backup created successfully: $backup_file ($backup_size bytes)"
                
                # Keep only last 5 backups
                cd /backups && ls -t backup-*.sql | tail -n +6 | xargs -r rm
                echo "Old backups cleaned up"
              else
                echo "Backup failed!"
                exit 1
              fi
              
              echo "Backup completed at $(date)"
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            volumeMounts:
            - name: backup-storage
              mountPath: /backups
            resources:
              requests:
                memory: "128Mi"
                cpu: "100m"
              limits:
                memory: "256Mi"
                cpu: "500m"
          volumes:
          - name: backup-storage
            emptyDir: {}
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
---
# CronJob for log cleanup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox:latest
            command: ['sh', '-c']
            args:
            - |
              echo "Starting log cleanup at $(date)"
              
              # Simulate log cleanup
              echo "Cleaning application logs older than 7 days..."
              find /logs -name "*.log" -type f -mtime +7 -exec rm {} \; 2>/dev/null || true
              
              echo "Cleaning temp files..."
              find /tmp -name "temp_*" -type f -mtime +1 -exec rm {} \; 2>/dev/null || true
              
              # Clean up old backups (if any)
              echo "Cleaning old backup files..."
              find /backups -name "backup-*.sql" -type f -mtime +30 -exec rm {} \; 2>/dev/null || true
              
              echo "Log cleanup completed at $(date)"
            volumeMounts:
            - name: log-storage
              mountPath: /logs
            - name: backup-storage
              mountPath: /backups
            resources:
              requests:
                memory: "64Mi"
                cpu: "50m"
              limits:
                memory: "128Mi"
                cpu: "100m"
          volumes:
          - name: log-storage
            emptyDir: {}
          - name: backup-storage
            emptyDir: {}
          restartPolicy: OnFailure
  suspend: false  # Set to true to suspend the cron job
---
# CronJob for metrics collection
apiVersion: batch/v1
kind: CronJob
metadata:
  name: metrics-collection
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: metrics-collector
            image: prom/blackbox-exporter:latest
            args:
            - '--config.file=/etc/blackbox_exporter/config.yml'
            - '--web.listen-address=:9115'
            - '--web.telemetry-path=/metrics'
            volumeMounts:
            - name: config
              mountPath: /etc/blackbox_exporter/
            resources:
              requests:
                memory: "64Mi"
                cpu: "50m"
              limits:
                memory: "128Mi"
                cpu: "100m"
          volumes:
          - name: config
            configMap:
              name: blackbox-exporter-config
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
---
# Blackbox Exporter ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: blackbox-exporter-config
data:
  config.yml: |
    modules:
      http_2xx:
        prober: http
        timeout: 5s
        http:
          method: GET
          path: /
          valid_http_versions: [ " "]
          valid_http_statuses: []  # Defaults to 2xx
      tcp_connect:
        prober: tcp
        timeout: 5s
---
# Test application with init container and sidecars
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-advanced-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-advanced-app
  template:
    metadata:
      labels:
        app: test-advanced-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: app-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumes:
      - name: app-config
        configMap:
          name: advanced-app-config
      - name: nginx-config
        configMap:
          name: nginx-advanced-config
```

**Deploy and Test Advanced Patterns:**
```powershell
# Deploy application with init containers and sidecars
kubectl apply -f 33-advanced-patterns.yaml

# Check pods
kubectl get pods -w

# Test application
# Open browser to http://localhost:30008
```

**Learning Objectives:**
- ✅ Understand init containers and their use cases
- ✅ Learn about sidecar containers and patterns
- ✅ Practice advanced deployment scenarios
- ✅ Monitor and troubleshoot complex applications

---

## Intermediate Summary

### ✅ **Core Concepts Mastered:**
1. **Ingress Controllers** - Manage external access
2. **Secrets Management** - Secure sensitive data
3. **RBAC** - Fine-grained access control
4. **Monitoring Stack** - Observability with Prometheus & Grafana
5. **ConfigMaps Advanced** - Complex configurations
6. **Init Containers & Sidecars** - Advanced pod patterns
7. **Jobs & CronJobs** - Batch processing and automation
8. **Network Policies** - Secure networking

### ✅ **Best Practices Learned:**
- Secure application design
- Observability and monitoring
- Declarative configuration management
- Automated deployment and scaling
- Advanced troubleshooting techniques

---

## Transition to Advanced Level

You're now ready for the **Advanced Level** with a solid foundation in:
- Core and advanced Kubernetes concepts
- Secure and production-ready application design
- Observability and monitoring best practices
- Declarative infrastructure and application management

The **Advanced Level** will build upon these concepts with:
1. **Custom Resource Definitions (CRDs)** - Extend Kubernetes capabilities
2. **Operators** - Manage complex stateful applications
3. **Service Mesh** - Advanced networking and security
4. **GitOps at Scale** - Manage multiple clusters and environments
5. **Kubernetes Security** - Network policies, PodSecurityPolicies
6. **Performance Tuning** - Optimize resource usage and application performance

Happy Learning! 🚀