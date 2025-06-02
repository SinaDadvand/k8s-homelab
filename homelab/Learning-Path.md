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
cd c:\Users\sinad\VS Code\homelab-prj\homelab

# Create the cluster using your configuration
# The kind-cluster.yaml includes extraPortMappings for NodePorts:
# - 30001 (sample-app)
# - 30002 (available for future use)  
# - 30003 (Helm chart)
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

**Why We Create This File:** A Pod is the smallest deployable unit in Kubernetes and represents one or more containers that share storage, network, and configuration. By creating this YAML file, we're defining a declarative specification for our first workload that Kubernetes will use to create and manage a running Pod instance. This file serves as your infrastructure-as-code document that can be version controlled and repeatedly applied.

**Create:** `01-nginx-pod.yaml`
```yaml
# Kubernetes API version for Pod resources - stable version for core objects
apiVersion: v1
# The type of Kubernetes resource we're defining
kind: Pod
# Metadata contains identifying information about the resource
metadata:
  # Unique name for this Pod within the namespace
  name: nginx-pod
  # Labels are key-value pairs used for identification and selection
  # The 'app: nginx' label helps us group and select this Pod later
  # The 'environment: learning' label categorizes this for educational purposes
  labels:
    app: nginx
    environment: learning
# Specification defines the desired state of the Pod
spec:
  # Array of containers that will run in this Pod
  containers:
  # Container definition - Pods can have multiple containers but we use one here
  - name: nginx  # Name of the container within the Pod
    # Docker image to run - 'latest' tag pulls the most recent version
    image: nginx:latest
    # Ports that the container exposes
    ports:
    # Container port 80 is where nginx serves HTTP traffic
    - containerPort: 80
    # Resource constraints to ensure fair resource allocation
    resources:
      # Minimum resources guaranteed to the container
      requests:
        memory: "64Mi"    # 64 Mebibytes of RAM guaranteed
        cpu: "50m"        # 50 millicores (0.05 CPU cores) guaranteed
      # Maximum resources the container can use
      limits:
        memory: "128Mi"   # 128 MiB maximum RAM usage
        cpu: "100m"       # 100 millicores (0.1 CPU cores) maximum
```

**Apply and Test:**
```powershell
# Apply the pod configuration to the cluster
# This command tells Kubernetes to create the pod based on our YAML specification
kubectl apply -f 01-nginx-pod.yaml

# Check pod status - verify the pod is running
# Shows basic information like STATUS, RESTARTS, and AGE
kubectl get pods

# Get detailed information about the pod including events and configuration
# This is crucial for troubleshooting if the pod doesn't start
kubectl describe pod nginx-pod

# View the container logs to see nginx startup messages
# Useful for debugging application issues
kubectl logs nginx-pod

# Access the pod directly by forwarding local port 8080 to container port 80
# This creates a tunnel from your machine to the pod
kubectl port-forward nginx-pod 8080:80
# Open browser to http://localhost:8080 to see the nginx welcome page
# Press Ctrl+C to stop port forwarding
```

**Learning Objectives:**
- ✅ Understand Pod structure
- ✅ Learn resource requests/limits
- ✅ Practice kubectl commands
- ✅ Understand port forwarding

### Step 4: Scale with Deployments

**Why We Create This File:** While Pods are great for understanding basic concepts, Deployments are what you'll use in production. A Deployment manages a set of identical Pods, providing declarative updates, scaling, and rolling updates. This YAML file defines not just one Pod, but a desired state of multiple replicas with automatic recovery if any Pod fails.

**Create:** `02-nginx-deployment.yaml`
```yaml
# apps/v1 API version is used for Deployment resources
apiVersion: apps/v1
# Deployment kind manages sets of Pods with scaling and rolling updates
kind: Deployment
metadata:
  # Name for this Deployment resource
  name: nginx-deployment
  # Labels help organize and select this Deployment
  labels:
    app: nginx
# Deployment specification
spec:
  # Number of Pod replicas to maintain
  replicas: 3
  # Selector defines which Pods this Deployment manages
  # Must match the labels in the Pod template below
  selector:
    matchLabels:
      app: nginx
  # Template defines the Pod specification for each replica
  template:
    # Metadata for each Pod created by this Deployment
    metadata:
      # Labels applied to each Pod - must match selector above
      labels:
        app: nginx
    # Pod specification (same as in Step 3, but with health checks)
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        # Resource constraints for each container
        resources:
          requests:
            memory: "64Mi"    # Guaranteed resources
            cpu: "50m"
          limits:
            memory: "128Mi"   # Maximum resources
            cpu: "100m"
        # Liveness probe checks if the container is still running
        # Kubernetes will restart the container if this fails
        livenessProbe:
          httpGet:
            path: /           # Check the root path
            port: 80          # On port 80
          initialDelaySeconds: 10  # Wait 10s before first check
          periodSeconds: 10        # Check every 10 seconds
        # Readiness probe checks if the container is ready to receive traffic
        # Kubernetes will not send traffic until this succeeds
        readinessProbe:
          httpGet:
            path: /           # Check the root path
            port: 80          # On port 80
          initialDelaySeconds: 5   # Wait 5s before first check
          periodSeconds: 5         # Check every 5 seconds
```

**Apply and Experiment:**
```powershell
# First, clean up the single pod from Step 3 since we're moving to Deployments
# Single pods don't auto-restart, but Deployment-managed pods do
kubectl delete pod nginx-pod

# Apply the deployment configuration to create multiple pod replicas
kubectl apply -f 02-nginx-deployment.yaml

# Watch pods being created in real-time (press Ctrl+C to stop watching)
# This shows how Kubernetes orchestrates multiple pod creation
kubectl get pods -w

# Check deployment status and see how many replicas are ready
kubectl get deployments

# Get detailed information about the deployment, including events
# Shows ReplicaSet creation and pod scheduling details
kubectl describe deployment nginx-deployment

# Demonstrate horizontal scaling - increase replicas to 5
# This shows how easy it is to scale applications in Kubernetes
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods

# Scale back down to 2 replicas to save resources
# Notice how Kubernetes gracefully terminates excess pods
kubectl scale deployment nginx-deployment --replicas=2
kubectl get pods
```

**Learning Objectives:**
- ✅ Understand Deployments vs Pods
- ✅ Learn about ReplicaSets
- ✅ Practice scaling applications
- ✅ Understand health checks (probes)

---

## Phase 3: Networking and Services

### Step 5: Expose Your Application

**Why We Create This File:** Services provide stable networking for your Pods. Unlike Pods which have ephemeral IP addresses that change when they restart, Services provide a consistent endpoint. This YAML creates a NodePort service that exposes your application both internally within the cluster and externally through a specific port.

**Create:** `03-nginx-service.yaml`
```yaml
# v1 API version for core Service resources
apiVersion: v1
# Service provides stable networking and load balancing for Pods
kind: Service
metadata:
  # Name for this Service - will become the DNS name inside the cluster
  name: nginx-service
# Service specification
spec:
  # Selector determines which Pods this Service will route traffic to
  # Must match the labels on the target Pods (from our Deployment)
  selector:
    app: nginx
  # Port configuration - Services can expose multiple ports
  ports:
  - name: http              # Name for this port (useful when multiple ports exist)
    port: 80                # Port the Service exposes inside the cluster
    targetPort: 80          # Port on the container to forward traffic to
    nodePort: 30002         # External port on each cluster node (30000-32767 range)
                           # This matches your kind-cluster.yaml port mapping
  # NodePort exposes the service on each node's IP at a static port
  # Other types: ClusterIP (internal only), LoadBalancer (cloud), ExternalName
  type: NodePort
```

**Apply and Test:**
```powershell
# Create the service to expose our nginx deployment
kubectl apply -f 03-nginx-service.yaml

# View all services in the cluster - notice the ClusterIP and NodePort
kubectl get services

# Get detailed service information including endpoints
# Endpoints show which Pod IPs the service will route traffic to
kubectl describe service nginx-service

# Test internal service discovery by creating a temporary pod
# This demonstrates how services work within the cluster
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- nginx-service

# Test external access from your host machine
# Thanks to KIND's port mapping in kind-cluster.yaml, this works
# Open browser to http://localhost:30002 to see nginx serving traffic from any of the pods
```

**Learning Objectives:**
- ✅ Understand Services and service discovery
- ✅ Learn about NodePort, ClusterIP service types
- ✅ Practice internal networking

### Step 6: Advanced Service Discovery

**Why We Create This File:** This demonstrates different service types and their use cases. ClusterIP services are for internal communication only, while headless services return individual Pod IPs instead of load balancing, useful for stateful applications that need direct Pod-to-Pod communication.

**Create:** `04-multiple-services.yaml`
```yaml
# ClusterIP Service - Default type, internal cluster access only
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  # Same selector as our NodePort service - targets the same Pods
  selector:
    app: nginx
  ports:
  - port: 80                # Port exposed within the cluster
    targetPort: 80          # Port on the target containers
  # ClusterIP is the default type - only accessible from within the cluster
  # Gets a stable internal IP address that doesn't change
  type: ClusterIP
---
# Headless Service - No load balancing, returns individual Pod IPs
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
  # Setting clusterIP to None makes this a "headless" service
  # DNS queries return all Pod IPs instead of a single service IP
  # Useful for stateful applications that need to know about all instances
  clusterIP: None
```

**Test Different Service Types:**
```powershell
# Create both ClusterIP and headless services
kubectl apply -f 04-multiple-services.yaml

# Test service discovery with DNS lookups
# Each service type behaves differently:

# 1. NodePort service returns a single IP (load balanced)
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup nginx-service

# 2. ClusterIP service also returns a single IP (load balanced)
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup nginx-clusterip

# 3. Headless service returns multiple IPs (one for each Pod)
# This is useful when you need to connect to specific Pod instances
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup nginx-headless

# Bonus: See all endpoints that services are routing to
kubectl get endpoints
```

---

## Phase 4: Configuration Management

### Step 7: ConfigMaps for Configuration

**Why We Create This File:** ConfigMaps separate configuration from application code, following the 12-factor app methodology. This allows you to modify configuration without rebuilding container images. We're creating both an nginx configuration file and a custom HTML page to demonstrate different ways to use ConfigMaps.

**Create:** `05-nginx-configmap.yaml`
```yaml
# ConfigMaps store non-confidential configuration data in key-value pairs
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
# Data section contains the configuration files
data:
  # Custom nginx configuration - replaces the default nginx.conf
  nginx.conf: |
    # Event processing configuration
    events {
        worker_connections 1024;    # Max connections per worker process
    }
    # HTTP server configuration
    http {
        server {
            listen 80;              # Listen on port 80
            # Main application location
            location / {
                root /usr/share/nginx/html;
                index index.html;
            }
            # Health check endpoint for monitoring and probes
            location /health {
                return 200 "OK\n";
                add_header Content-Type text/plain;
            }
        }
    }
  # Custom HTML content - will replace the default nginx page
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Learning Kubernetes!</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            h1 { color: #326ce5; }
        </style>
    </head>
    <body>
        <h1>Hello from Kubernetes!</h1>
        <p>This content is served from a ConfigMap</p>
        <p><strong>Pod:</strong> ${HOSTNAME}</p>
        <p><em>Configuration as Code in Action!</em></p>
    </body>
    </html>
```

**Why We Create This File:** This demonstrates how to consume ConfigMaps in a Deployment. We mount configuration files as volumes and inject environment variables, showing two common patterns for using configuration data in Kubernetes applications.

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
        app: nginx-config      # Different label to distinguish from previous deployment
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        # Volume mounts define where to mount the ConfigMap data inside the container
        volumeMounts:
        # Mount the custom nginx.conf file to replace the default configuration
        - name: nginx-config-volume
          mountPath: /etc/nginx/nginx.conf    # Destination path in container
          subPath: nginx.conf                 # Specific key from ConfigMap
        # Mount the custom HTML file to replace the default nginx page
        - name: html-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        # Environment variables can be injected from various sources
        env:
        # Get the pod name and make it available as HOSTNAME environment variable
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name        # Reference to pod metadata
      # Volumes section defines the data sources to mount
      volumes:
      # Volume sourced from our nginx-config ConfigMap
      - name: nginx-config-volume
        configMap:
          name: nginx-config                  # Must match the ConfigMap name
      # Same ConfigMap used for HTML content (demonstrates reuse)
      - name: html-volume
        configMap:
          name: nginx-config
```

**Apply and Test:**
```powershell
# First create the ConfigMap with our custom configuration
kubectl apply -f 05-nginx-configmap.yaml

# Then deploy the application that uses the ConfigMap
kubectl apply -f 06-nginx-with-config.yaml

# Create a service to expose the new deployment
# Using kubectl expose command instead of YAML for variety
kubectl expose deployment nginx-with-config --port=80 --type=NodePort

# Check the services and note the new NodePort assigned
kubectl get services

# Test the custom configuration by accessing the new service
# The HTML should show our custom content from the ConfigMap
# Open browser to the NodePort shown above (e.g., http://localhost:3xxxx)

# Verify the custom health endpoint works
kubectl get service nginx-with-config  # Note the NodePort
# Then test: curl http://localhost:[NodePort]/health

# Check that environment variables are properly injected
kubectl exec deployment/nginx-with-config -- env | grep HOSTNAME
```

**Learning Objectives:**
- ✅ Understand ConfigMaps
- ✅ Learn volume mounting
- ✅ Practice environment variables
- ✅ Understand configuration separation

---

## Phase 5: Debugging and Troubleshooting (Busybox)

### Step 8: Debugging Tools

**Why We Create This File:** BusyBox is a Swiss Army knife for debugging Kubernetes clusters. It contains many common Unix utilities in a tiny package, making it perfect for troubleshooting networking, DNS resolution, and connectivity issues within your cluster.

**Create:** `07-debug-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  # Labels help organize and identify debugging pods
  labels:
    purpose: debugging
    tool: busybox
spec:
  containers:
  - name: busybox
    # BusyBox image contains many useful debugging tools
    image: busybox:latest
    # Keep the container running so we can exec into it
    command: ['sleep', '3600']    # Sleep for 1 hour
    # Minimal resource allocation since this is just for debugging
    resources:
      requests:
        memory: "32Mi"    # 32 Mebibytes - very lightweight
        cpu: "10m"        # 10 millicores - minimal CPU
      limits:
        memory: "64Mi"    # Maximum 64 MiB
        cpu: "50m"        # Maximum 50 millicores
  # OnFailure restart policy is appropriate for debugging pods
  restartPolicy: OnFailure
```

**Debugging Commands:**
```powershell
# Deploy the debug pod
kubectl apply -f 07-debug-pod.yaml

# Wait for the pod to be ready
kubectl wait --for=condition=ready pod/debug-pod --timeout=60s

# Execute an interactive shell inside the debug pod
# This gives you a command prompt inside the cluster
kubectl exec -it debug-pod -- /bin/sh

# ===== COMMANDS TO RUN INSIDE THE DEBUG POD =====
# Test HTTP connectivity to our nginx service
wget -qO- nginx-service

# Test DNS resolution - should show service IP
nslookup nginx-service

# Test DNS resolution for different service types
nslookup nginx-clusterip
nslookup nginx-headless    # Should return multiple IPs

# Test network connectivity (may not work if ICMP is blocked)
ping nginx-service

# View Kubernetes environment variables automatically injected
env | grep KUBERNETES

# Check if specific ports are open
telnet nginx-service 80

# View network interfaces and routing
ip addr show
ip route show

# Test specific endpoints
wget -qO- nginx-service/health
# Exit the pod shell with: exit

# ===== COMMANDS TO RUN FROM YOUR HOST =====
# From outside the pod, check pod details and events
kubectl describe pod debug-pod

# View logs from the debug pod (usually minimal for BusyBox)
kubectl logs debug-pod

# Check cluster events for troubleshooting
kubectl get events --sort-by=.metadata.creationTimestamp

# Test DNS from outside the cluster
kubectl exec debug-pod -- nslookup kubernetes.default.svc.cluster.local
```

### Step 9: Troubleshooting Common Issues

### Why We Create This File:
The `08-broken-pod.yaml` file demonstrates common failure scenarios you'll encounter in production Kubernetes environments. By intentionally creating broken configurations, we learn systematic troubleshooting approaches that are essential for maintaining applications in real-world deployments. This practice helps build muscle memory for debugging workflows.

**Create:** `08-broken-pod.yaml` (Intentionally broken for learning)
```yaml
# This Pod demonstrates common failure patterns in Kubernetes deployments
apiVersion: v1        # Using the core v1 API for basic Pod resources
kind: Pod            # Defining a single Pod (not managed by a controller)
metadata:
  name: broken-pod   # Simple descriptive name for our troubleshooting exercise
spec:
  containers:        # Container specifications - this is where our intentional error lives
  - name: broken-container           # Container name for identification in logs
    image: nginx:nonexistent-tag     # INTENTIONAL ERROR: This tag doesn't exist, causing ImagePullBackOff
    ports:
    - containerPort: 80              # Standard HTTP port for nginx (won't matter since container won't start)
```

**Comprehensive Troubleshooting Practice:**
```powershell
# Apply the broken configuration to see failure in action
kubectl apply -f 08-broken-pod.yaml
# Expected output: pod/broken-pod created

# Step 1: Check overall pod status - this gives you the high-level view
kubectl get pods
# Look for: broken-pod   0/1     ImagePullBackOff   0          2m

# Step 2: Get detailed pod information - this shows events and detailed status
kubectl describe pod broken-pod
# Key sections to examine:
# - Events: Shows the sequence of what Kubernetes tried to do
# - State: Shows current container state (Waiting, Running, Terminated)
# - Reason: Explains why the container is in its current state

# Step 3: Attempt to get container logs (will fail but shows the process)
kubectl logs broken-pod
# Expected: Error from server (BadRequest): container "broken-container" in pod "broken-pod" is waiting to start: trying and failing to pull image

# Step 4: Get more detailed events across the cluster
kubectl get events --sort-by=.metadata.creationTimestamp
# This shows cluster-wide events, useful for understanding timing and sequences

# Step 5: Fix the issue by updating the image
kubectl delete pod broken-pod
# Always clean up before applying fixes

# Create a corrected version or edit the file to use nginx:latest
# Then reapply
kubectl apply -f 08-broken-pod.yaml
# Now it should work: pod/broken-pod created

# Step 6: Verify the fix worked
kubectl get pods
# Should show: broken-pod   1/1     Running   0          30s

# Step 7: Test that the application actually works
kubectl port-forward pod/broken-pod 8080:80
# In another terminal: curl http://localhost:8080
```

**Additional Debugging Scenarios to Practice:**

```powershell
# Scenario 1: Resource constraints causing eviction
# Create a pod that requests too much memory
kubectl run memory-hog --image=nginx --requests='memory=10Gi'
kubectl describe pod memory-hog
# Look for: FailedScheduling events

# Scenario 2: Misconfigured environment variables
kubectl run env-test --image=nginx --env="MYSQL_HOST=nonexistent-service"
kubectl logs env-test

# Create a pod that tries to connect to a database
kubectl run db-test --image=postgres:15 --env="POSTGRES_HOST=nonexistent-db"
kubectl logs db-test

# Practice reading application logs for configuration errors

# Scenario 3: Network connectivity issues
kubectl run network-test --image=busybox --command -- sleep 3600
kubectl exec -it network-test -- nslookup kubernetes.default
# Test DNS resolution and network connectivity from inside pods

# Clean up all test pods
kubectl delete pod broken-pod memory-hog env-test network-test --ignore-not-found=true
```

**Learning Objectives:**
- ✅ Master kubectl exec for debugging
- ✅ Learn log analysis
- ✅ Practice troubleshooting workflows
- ✅ Understand common failure patterns

---

## Phase 6: Persistent Storage (Redis)

### Step 10: Stateless vs Stateful Applications

### Why We Create This File:
This Redis deployment demonstrates the difference between stateless and stateful applications. Most applications we've worked with so far are stateless - they don't store critical data locally. Redis is a stateful application that stores data in memory. By deploying Redis without persistent storage, we can observe how container restarts cause data loss, highlighting the need for persistent storage solutions.

**Create:** `09-redis-deployment.yaml`
```yaml
# Redis deployment without persistent storage - data will be lost on pod restart
apiVersion: apps/v1      # apps/v1 API for Deployment resources
kind: Deployment         # Using Deployment instead of StatefulSet (we'll see the difference)
metadata:
  name: redis-deployment
spec:
  replicas: 1            # Only one replica since we're not using shared storage
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis       # Labels for pod identification and service selection
    spec:
      containers:
      - name: redis
        image: redis:alpine     # Alpine version for smaller footprint
        ports:
        - containerPort: 6379   # Standard Redis port
        # Resource constraints appropriate for Redis workload
        resources:
          requests:
            memory: "64Mi"      # Redis needs memory for data storage
            cpu: "50m"          # Minimal CPU for basic operations
          limits:
            memory: "128Mi"     # Prevent Redis from consuming too much memory
            cpu: "100m"         # Maximum CPU allocation
```

**Test Data Persistence (Demonstrating Data Loss):**
```powershell
# Deploy Redis without persistent storage
kubectl apply -f 09-redis-deployment.yaml

# Wait for Redis to be ready
kubectl wait --for=condition=available --timeout=60s deployment/redis-deployment

# Expose Redis service for internal cluster access
kubectl expose deployment redis-deployment --port=6379 --type=ClusterIP

# Connect to Redis and add some test data
kubectl exec -it deployment/redis-deployment -- redis-cli
# ===== COMMANDS TO RUN INSIDE REDIS CLI =====
# SET mykey "Hello Kubernetes"
# SET user:1 "Alice"
# SET user:2 "Bob"
# LPUSH logs "Application started"
# LPUSH logs "User logged in"
# GET mykey
# LRANGE logs 0 -1
# exit
# ===== END REDIS CLI COMMANDS =====

# Verify our data exists
kubectl exec -it deployment/redis-deployment -- redis-cli GET mykey
# Should return: "Hello Kubernetes"

# Now simulate a pod failure by deleting the pod
kubectl delete pod -l app=redis

# Watch the pod restart automatically (Deployment ensures replica count)
kubectl get pods -w
# Wait for the new pod to be Running and Ready

# Test if our data survived the restart (it won't!)
kubectl exec -it deployment/redis-deployment -- redis-cli GET mykey
# Expected result: (nil) - data is gone because Redis stored it in container's ephemeral storage

# Try to get the logs we added
kubectl exec -it deployment/redis-deployment -- redis-cli LRANGE logs 0 -1
# Expected result: (empty list) - all data is lost
```

### Step 11: Persistent Volumes

### Why We Create This File:
This configuration solves the data persistence problem we observed in Step 10. By using a PersistentVolumeClaim (PVC) and StatefulSet instead of a Deployment, we ensure that Redis data survives pod restarts and deletions. StatefulSets are designed for stateful applications that need stable network identity and persistent storage.

**Create:** `10-redis-persistent.yaml`
```yaml
# PersistentVolumeClaim requests storage from the cluster
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc           # Name to reference this claim in StatefulSet
spec:
  accessModes:
  - ReadWriteOnce           # Volume can be mounted read-write by a single node
                           # Other modes: ReadOnlyMany, ReadWriteMany
  resources:
    requests:
      storage: 1Gi          # Request 1 Gigabyte of storage
                           # In production, size this based on expected data volume
---
# StatefulSet provides guarantees about ordering and uniqueness of pods
# Unlike Deployments, StatefulSet maintains sticky identity for each pod
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-statefulset
spec:
  serviceName: redis-headless    # Required: name of headless service for network identity
  replicas: 1                    # Single replica for this learning example
  selector:
    matchLabels:
      app: redis-persistent      # Different label from stateless version
  template:
    metadata:
      labels:
        app: redis-persistent    # Must match selector above
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        # Volume mount configuration - critical for data persistence
        volumeMounts:
        - name: redis-storage         # Name must match volume definition below
          mountPath: /data            # Redis default data directory
        # Redis configuration for persistence
        command: ["redis-server", "--appendonly", "yes"]
        # --appendonly yes enables Redis AOF (Append Only File) persistence
        # This writes every write operation to a log file for durability
        resources:
          requests:
            memory: "64Mi"            # Minimum memory for Redis operations
            cpu: "50m"                # Minimal CPU requirement
          limits:
            memory: "128Mi"           # Maximum memory to prevent OOM
            cpu: "100m"               # CPU limit for resource sharing
      # Volumes section defines storage sources
      volumes:
      - name: redis-storage          # Volume name referenced in volumeMounts
        persistentVolumeClaim:
          claimName: redis-pvc       # References the PVC defined above
---
# Service for accessing Redis
apiVersion: v1
kind: Service
metadata:
  name: redis-persistent-service
spec:
  selector:
    app: redis-persistent           # Routes traffic to StatefulSet pods
  ports:
  - port: 6379                      # Service port
    targetPort: 6379                # Container port
  type: ClusterIP                   # Internal access only
---
# Headless service required by StatefulSet for pod network identity
apiVersion: v1
kind: Service
metadata:
  name: redis-headless             # Must match serviceName in StatefulSet spec
spec:
  clusterIP: None                  # Makes this a headless service
  selector:
    app: redis-persistent
  ports:
  - port: 6379
    targetPort: 6379
```

**Test Persistent Storage (Proving Data Survives):**
```powershell
# First, clean up the stateless Redis deployment
kubectl delete deployment redis-deployment
kubectl delete service redis-deployment

# Apply the persistent storage configuration
kubectl apply -f 10-redis-persistent.yaml

# Wait for the StatefulSet to be ready and PVC to be bound
kubectl wait --for=condition=ready --timeout=300s pod/redis-statefulset-0
kubectl get statefulsets
kubectl get pvc
# You should see: redis-pvc   Bound    pvc-xxxxx   1Gi

# Connect to Redis and add persistent data
kubectl exec -it redis-statefulset-0 -- redis-cli
# ===== COMMANDS TO RUN INSIDE REDIS CLI =====
# SET persistent-key "This data will survive pod restarts!"
# SET user:persistent:1 "Alice (persistent)"
# SET user:persistent:2 "Bob (persistent)"
# LPUSH activity-log "User session started"
# LPUSH activity-log "Data saved to persistent storage"
# HSET app:config version "2.0"
# HSET app:config environment "production"
# GET persistent-key
# LRANGE activity-log 0 -1
# HGETALL app:config
# exit
# ===== END REDIS CLI COMMANDS =====

# Now test persistence by deleting the pod (this simulates a crash)
kubectl delete pod redis-statefulset-0

# Watch StatefulSet automatically recreate the pod with the same name
kubectl get pods -w
# Wait for redis-statefulset-0 to be Running and Ready

# Verify our data survived the pod deletion and recreation
kubectl exec -it redis-statefulset-0 -- redis-cli GET persistent-key
# Should return: "This data will survive pod restarts!"

# Check all our saved data
kubectl exec -it redis-statefulset-0 -- redis-cli LRANGE activity-log 0 -1
kubectl exec -it redis-statefulset-0 -- redis-cli HGETALL app:config

# Test with more data to prove persistence is working
kubectl exec -it redis-statefulset-0 -- redis-cli
# ===== ADDITIONAL TEST COMMANDS =====
# SET test-after-restart "Added after pod restart"
# LPUSH activity-log "Post-restart activity"
# GET persistent-key
# GET test-after-restart
# LRANGE activity-log 0 -1
# exit
# ===== END TEST COMMANDS =====

# Bonus: Examine the persistent volume
kubectl describe pvc redis-pvc
kubectl get pv
# This shows the underlying persistent volume created by your cluster
```

**Learning Objectives:**
- ✅ Understand StatefulSets vs Deployments
- ✅ Learn Persistent Volumes and Claims
- ✅ Practice data persistence
- ✅ Understand stateful application patterns

---

## Phase 7: Package Management with Helm

### Step 13: Introduction to Helm

### Why We Learn Helm:
Helm is the package manager for Kubernetes, often called "the apt/yum for Kubernetes." Instead of managing dozens of YAML files manually, Helm allows you to package, version, and deploy applications as reusable charts. This dramatically simplifies application lifecycle management and enables templating for different environments.

**Install Helm:**
```powershell
# Install Helm using Windows Package Manager
winget install Helm.Helm

# Verify Helm installation and check version
helm version
# Expected output shows both client and server versions

# Add popular Helm repositories for pre-built charts
helm repo add stable https://charts.helm.sh/stable       # Stable charts (community)
helm repo add bitnami https://charts.bitnami.com/bitnami # Bitnami charts (production-ready)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update repository index to get latest chart versions
helm repo update

# List available repositories
helm repo list

# Search for available charts (example searches)
helm search repo nginx          # Find nginx-related charts
helm search repo redis          # Find redis charts
helm search repo wordpress      # Find WordPress charts
```

### Step 14: Your First Helm Chart

### Why We Create This Chart:
Creating your own Helm chart teaches you how to template Kubernetes manifests and make them reusable across different environments. You'll learn about Helm's templating engine, value injection, and how to structure a professional-grade chart that can be shared and versioned.

**Create Your First Chart:**
```powershell
# Create a new Helm chart with standard structure
helm create my-nginx-chart

# Navigate to the chart directory to explore the structure
cd my-nginx-chart

# Examine the chart structure (Helm creates a standard layout)
Get-ChildItem -Recurse
# You'll see:
# Chart.yaml      - Chart metadata (name, version, description)
# values.yaml     - Default configuration values
# templates/      - Kubernetes manifest templates
# charts/         - Dependency charts (if any)
# .helmignore     - Files to ignore when packaging

# Look at the generated files to understand the structure
Get-Content Chart.yaml      # Chart metadata
Get-Content values.yaml     # Default values for templating
```

**Customize the Chart Values:**

**Edit:** `my-nginx-chart/values.yaml`
```yaml
# Default values for my-nginx-chart - these can be overridden during installation
replicaCount: 2              # Number of pod replicas to create

# Container image configuration
image:
  repository: nginx          # Docker image repository
  pullPolicy: IfNotPresent   # Image pull policy (Always, Never, IfNotPresent)
  tag: "latest"             # Image tag to use

# Service Account configuration
serviceAccount:
  create: true              # Create a service account
  automount: true           # Automatically mount service account token
  annotations: {}           # Annotations to add to the service account
  name: ""                  # The name of the service account (if empty, uses fullname template)

# Service configuration  
service:
  type: NodePort            # Service type (ClusterIP, NodePort, LoadBalancer)
  port: 80                  # Port the service exposes
  nodePort: 30003          # Specific NodePort (matches your kind-cluster.yaml mapping)

# Ingress configuration (disabled for this learning example)
ingress:
  enabled: false            # Set to true to enable ingress

# Resource limits and requests for containers
resources:
  limits:
    cpu: 100m               # Maximum CPU (100 millicores)
    memory: 128Mi           # Maximum memory (128 Mebibytes)
  requests:
    cpu: 50m                # Guaranteed CPU
    memory: 64Mi            # Guaranteed memory

# Autoscaling configuration (disabled by default)
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# Node selection and tolerations (empty by default)
nodeSelector: {}
tolerations: []
affinity: {}

# Custom application configuration
appConfig:
  environment: "development"  # Custom value we'll use in templates
  debug: true                # Custom debug flag
  version: "1.0.0"          # Application version
```

**Customize the Deployment Template:**

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
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "my-nginx-chart.selectorLabels" . | nindent 8 }}
        environment: {{ .Values.appConfig.environment }}
        version: {{ .Values.appConfig.version }}
    spec:
      containers:
                - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          env:
            - name: APP_ENV
              value: {{ .Values.appConfig.environment | quote }}
            - name: DEBUG
              value: {{ .Values.appConfig.debug | quote }}
            - name: APP_VERSION
              value: {{ .Values.appConfig.version | quote }}
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          volumeMounts:
            - name: config
              mountPath: /usr/share/nginx/html/config.json
              subPath: config.json
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      volumes:
        - name: config
          configMap:
            name: {{ include "my-nginx-chart.fullname" . }}-config
```

**Create a ConfigMap Template:**

**Create:** `my-nginx-chart/templates/configmap.yaml`
```yaml
# ConfigMap template demonstrating complex templating
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-nginx-chart.fullname" . }}-config
  labels:
    {{- include "my-nginx-chart.labels" . | nindent 4 }}
data:
  # JSON configuration generated from Helm values
  config.json: |
    {
      "application": {
        "name": "{{ include "my-nginx-chart.fullname" . }}",
        "version": "{{ .Values.appConfig.version }}",
        "environment": "{{ .Values.appConfig.environment }}",
        "debug": {{ .Values.appConfig.debug }},
        "replicas": {{ .Values.replicaCount }},
        "chart": {
          "name": "{{ .Chart.Name }}",
          "version": "{{ .Chart.Version }}",
          "appVersion": "{{ .Chart.AppVersion }}"
        }
      },
      "kubernetes": {
        "namespace": "{{ .Release.Namespace }}",
        "release": "{{ .Release.Name }}",
        "service": "{{ .Release.Service }}"
      }
    }
  # HTML page showing configuration
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>{{ include "my-nginx-chart.fullname" . }}</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            h1 { color: #326ce5; }
            .config { background: #f5f5f5; padding: 20px; border-radius: 5px; }
        </style>
    </head>
    <body>
        <h1>Helm Chart Application</h1>
        <p><strong>Application:</strong> {{ include "my-nginx-chart.fullname" . }}</p>
        <p><strong>Version:</strong> {{ .Values.appConfig.version }}</p>
        <p><strong>Environment:</strong> {{ .Values.appConfig.environment }}</p>
        <p><strong>Debug Mode:</strong> {{ .Values.appConfig.debug }}</p>
        <p><strong>Replicas:</strong> {{ .Values.replicaCount }}</p>
        
        <h2>Chart Information</h2>
        <div class="config">
            <p><strong>Chart Name:</strong> {{ .Chart.Name }}</p>
            <p><strong>Chart Version:</strong> {{ .Chart.Version }}</p>
            <p><strong>App Version:</strong> {{ .Chart.AppVersion }}</p>
            <p><strong>Release Name:</strong> {{ .Release.Name }}</p>
            <p><strong>Namespace:</strong> {{ .Release.Namespace }}</p>
        </div>
        
        <h2>Links</h2>
        <ul>
            <li><a href="/config.json">View JSON Config</a></li>
        </ul>
    </body>
    </html>
```

**Update the Service Template:**

**Edit:** `my-nginx-chart/templates/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-nginx-chart.fullname" . }}
  labels:
    {{- include "my-nginx-chart.labels" . | nindent 4 }}
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "80"
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if eq .Values.service.type "NodePort" }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}  selector:
    {{- include "my-nginx-chart.selectorLabels" . | nindent 4 }}
```

**Add Missing Helper Function:**

**Edit:** `my-nginx-chart/templates/_helpers.tpl`
Add the following helper function at the end of the file:
```yaml
{{/*
Create the name of the service account to use
*/}}
{{- define "my-nginx-chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-nginx-chart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Step 15: Deploy and Manage with Helm

**Deploy Your Chart:**
```powershell
# Navigate back to the main directory (out of chart folder)
cd ..

# Validate the chart before installation (dry-run)
helm template my-nginx-release my-nginx-chart --debug

# Install the chart with default values
helm install my-nginx-release my-nginx-chart

# Check the Helm release status
helm list

# Check all resources created by this Helm release
kubectl get all -l app.kubernetes.io/instance=my-nginx-release

# View the generated manifests that were actually applied
helm get manifest my-nginx-release

# Check the values that were used during installation
helm get values my-nginx-release

# Test the deployment
helm test my-nginx-release

# Access the application
curl http://localhost:30003
# Access the custom config endpoint
curl http://localhost:30003/config.json
```

**Manage Helm Releases:**
```powershell
# Upgrade the release with new values (change replica count)
helm upgrade my-nginx-release my-nginx-chart --set replicaCount=4

# Check the upgrade
kubectl get pods -l app.kubernetes.io/instance=my-nginx-release

# Create custom values file for more complex changes
@"
replicaCount: 3
appConfig:
  environment: "staging"
  debug: false
  version: "2.0.0"
"@ | Out-File -FilePath custom-values.yaml -Encoding UTF8

# Apply the custom values
helm upgrade my-nginx-release my-nginx-chart -f custom-values.yaml

# View release history
helm history my-nginx-release

# Rollback to previous version if needed
helm rollback my-nginx-release 1

# Check rollback worked
helm history my-nginx-release

# Uninstall the release (removes all resources)
helm uninstall my-nginx-release

# Verify cleanup
kubectl get all -l app.kubernetes.io/instance=my-nginx-release

# Clean up the custom values file
Remove-Item custom-values.yaml -ErrorAction SilentlyContinue
```



### Step 16: Using Public Helm Charts

**Why We Create This:** Learn to leverage the vast ecosystem of pre-built Helm charts from public repositories. This demonstrates how to quickly deploy complex applications like databases, web servers, and full application stacks without writing manifests from scratch. Understanding public charts is essential for production Kubernetes deployments where you want to use battle-tested, community-maintained configurations.

**Deploy Redis using Bitnami Chart:**
```powershell
# Search for Redis charts in all added repositories
# This shows available Redis charts with their versions and descriptions
helm search repo redis

# Search for charts in the Artifact Hub (comprehensive chart registry)
# This finds charts across all public repositories, not just locally added ones
helm search hub redis

# Get detailed information about the Bitnami Redis chart
# Shows chart description, version, app version, and maintainers
helm show chart bitnami/redis

# View the default values for the Redis chart
# This shows all configurable parameters and their default values
helm show values bitnami/redis

# Install Redis using Bitnami chart with custom configurations
# --set allows you to override default values without creating a values file
# auth.enabled=false: Disables Redis password authentication for easier testing
# master.persistence.enabled=false: Disables persistent storage (data will be lost on restart)
# replica.persistence.enabled=false: Disables persistence for replica nodes too
helm install my-redis bitnami/redis \
  --set auth.enabled=false \
  --set master.persistence.enabled=false \
  --set replica.persistence.enabled=false \
  --set replica.replicaCount=1

# Alternative: Install with a custom values file for more complex configurations
# Create a custom-redis-values.yaml file first (see below)
# helm install my-redis bitnami/redis -f custom-redis-values.yaml

# Check all Kubernetes resources created by this Helm release
# The label app.kubernetes.io/instance=my-redis is automatically added by Helm
kubectl get all -l app.kubernetes.io/instance=my-redis

# Get detailed information about the Helm release
# Shows release status, deployed chart version, and notes for connecting to Redis
helm status my-redis

# List all Helm releases in the current namespace
helm list

# Check the Redis pods are running and view their logs
kubectl get pods -l app.kubernetes.io/name=redis
kubectl logs -l app.kubernetes.io/name=redis -f

# Test Redis connection using a temporary client pod
# --rm: Remove pod after exit
# --tty: Allocate a TTY for interactive session
# -i: Keep STDIN open for interactive session
# --restart='Never': Don't restart the pod if it fails
kubectl run redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:7.0-debian-11 -- bash

# Inside the Redis client pod, connect to the Redis master service:
# redis-cli -h my-redis-master
# SET test-key "Hello from Helm Redis"
# GET test-key
# KEYS *
# INFO replication
# exit

# Test Redis performance (run this outside the interactive session)
kubectl run redis-benchmark --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:7.0-debian-11 -- \
  redis-benchmark -h my-redis-master -c 10 -n 1000

# View Redis configuration
kubectl exec -it deployment/my-redis-master -- cat /opt/bitnami/redis/etc/redis.conf
```

**Create custom Redis values file:**

**Create:** `custom-redis-values.yaml`
```yaml
# Custom Redis configuration demonstrating various Helm chart customization options

# Global configuration that applies to all Redis components
global:
  # Image registry for all Redis images
  imageRegistry: "docker.io"
  # Image pull policy for all containers
  imagePullPolicy: IfNotPresent
  # Storage class for persistent volumes (if persistence enabled)
  storageClass: ""

# Redis master configuration
master:
  # Number of master replicas (should always be 1 for Redis)
  count: 1
  
  # Resource requests and limits for the master pod
  resources:
    requests:
      memory: "256Mi"      # Minimum memory allocation
      cpu: "100m"          # Minimum CPU allocation (0.1 CPU core)
    limits:
      memory: "512Mi"      # Maximum memory allocation
      cpu: "500m"          # Maximum CPU allocation (0.5 CPU core)
  
  # Persistence configuration for master
  persistence:
    enabled: false          # Disable persistent storage for learning purposes
    size: "1Gi"            # Size of persistent volume if enabled
    accessModes:
      - "ReadWriteOnce"    # Volume can be mounted read-write by single node
  
  # Redis configuration parameters
  configuration: |-
    # Redis server configuration
    maxmemory 256mb                    # Maximum memory usage
    maxmemory-policy allkeys-lru       # Eviction policy when memory limit reached
    save 900 1                         # Save snapshot if at least 1 key changed in 900 seconds
    save 300 10                        # Save snapshot if at least 10 keys changed in 300 seconds
    save 60 10000                      # Save snapshot if at least 10000 keys changed in 60 seconds

# Redis replica configuration
replica:
  # Number of read-only replica nodes
  replicaCount: 1
  
  # Resource allocation for replica pods
  resources:
    requests:
      memory: "128Mi"
      cpu: "50m"
    limits:
      memory: "256Mi"
      cpu: "250m"
  
  # Persistence for replicas (usually disabled as they sync from master)
  persistence:
    enabled: false

# Authentication settings
auth:
  enabled: false            # Disable password authentication for easier learning
  # password: "mypassword"  # Uncomment to set a specific password

# Service configuration
service:
  type: ClusterIP          # Internal service type (ClusterIP, NodePort, LoadBalancer)
  ports:
    redis: 6379            # Redis port number

# Metrics and monitoring configuration
metrics:
  enabled: true            # Enable Redis metrics exporter for Prometheus
  serviceMonitor:
    enabled: false         # Enable Prometheus ServiceMonitor (requires Prometheus Operator)
  
  # Resources for metrics exporter sidecar
  resources:
    requests:
      memory: "32Mi"
      cpu: "10m"
    limits:
      memory: "64Mi"
      cpu: "50m"

# Network policy configuration (advanced security)
networkPolicy:
  enabled: false           # Disable network policies for simplicity
  allowExternal: true      # Allow external traffic if network policies enabled

# Pod security context
securityContext:
  enabled: true
  fsGroup: 1001           # Group ID for filesystem permissions
  runAsUser: 1001         # User ID to run Redis processes

# Additional labels and annotations
commonLabels:
  environment: "learning"  # Custom label for all resources
  project: "homelab"

commonAnnotations:
  managed-by: "helm"       # Custom annotation for all resources
```

**Deploy WordPress with MySQL:**
```powershell
# View available WordPress charts and their information
helm search repo wordpress
helm show chart bitnami/wordpress

# Inspect WordPress default values to understand configuration options
# This shows database settings, admin credentials, service configuration, etc.
helm show values bitnami/wordpress | head -50

# Install WordPress with MySQL backend using custom configuration
# service.type=NodePort: Expose WordPress via NodePort for external access
# service.nodePorts.http=30004: Specific port number for access (http://localhost:30004)
# wordpressUsername/Password: Admin credentials for WordPress dashboard
# mariadb.primary.persistence.enabled=false: Disable MySQL data persistence for learning
# wordpressBlogName: Custom blog title
# wordpressEmail: Admin email address
helm install my-wordpress bitnami/wordpress \
  --set service.type=NodePort \
  --set service.nodePorts.http=30004 \
  --set wordpressUsername=admin \
  --set wordpressPassword=password123 \
  --set wordpressBlogName="My K8s Learning Blog" \
  --set wordpressEmail="admin@homelab.local" \
  --set mariadb.primary.persistence.enabled=false \
  --set mariadb.auth.database=wordpress \
  --set mariadb.auth.username=wpuser

# Monitor the deployment progress
# WordPress requires both the web server and MySQL database to be ready
kubectl get pods -l app.kubernetes.io/instance=my-wordpress -w

# Check all resources created by the WordPress Helm chart
# This includes deployments, services, secrets, configmaps, and persistent volume claims
kubectl get all,secrets,configmaps,pvc -l app.kubernetes.io/instance=my-wordpress

# View the WordPress deployment details
kubectl describe deployment my-wordpress

# Check the MariaDB database deployment
kubectl describe deployment my-wordpress-mariadb

# Get detailed Helm release information
# Shows release status, deployed resources, and connection instructions
helm status my-wordpress

# Check WordPress pod logs for any startup issues
kubectl logs -l app.kubernetes.io/name=wordpress -f

# Check MariaDB pod logs
kubectl logs -l app.kubernetes.io/name=mariadb -f

# Test WordPress accessibility
echo "WordPress is accessible at: http://localhost:30004"
echo "Admin Dashboard: http://localhost:30004/wp-admin/"
echo "Username: admin"
echo "Password: password123"

# Open browser programmatically (optional)
# start http://localhost:30004

# Test database connectivity from WordPress pod
kubectl exec -it deployment/my-wordpress -- wp db check

# View WordPress configuration
kubectl exec -it deployment/my-wordpress -- cat /opt/bitnami/wordpress/wp-config.php

# Create a backup of WordPress data (if needed)
kubectl exec -it deployment/my-wordpress -- wp db export /tmp/wordpress-backup.sql

# Scale WordPress (demonstrates multi-pod WordPress limitations without shared storage)
helm upgrade my-wordpress bitnami/wordpress \
  --set replicaCount=2 \
  --reuse-values

# Verify scaling
kubectl get pods -l app.kubernetes.io/name=wordpress

# Clean up resources when done with testing
# This removes all Kubernetes resources created by these Helm releases
helm uninstall my-wordpress
helm uninstall my-redis

# Verify cleanup
kubectl get all -l app.kubernetes.io/instance=my-wordpress
kubectl get all -l app.kubernetes.io/instance=my-redis
```

**Create custom WordPress values file (optional advanced configuration):**

**Create:** `custom-wordpress-values.yaml`
```yaml
# Advanced WordPress configuration with comprehensive customization options

# Global settings
global:
  imageRegistry: "docker.io"
  imagePullPolicy: IfNotPresent

# WordPress application configuration
image:
  registry: docker.io
  repository: bitnami/wordpress
  tag: "6.3.1-debian-11-r0"

# WordPress admin configuration
wordpressUsername: admin
wordpressPassword: SecurePassword123!
wordpressEmail: admin@homelab.local
wordpressBlogName: "Kubernetes Learning Blog"
wordpressFirstName: "K8s"
wordpressLastName: "Administrator"

# WordPress application settings
wordpressScheme: http
wordpressSkipInstall: false        # Set to true to skip WordPress installation wizard
wordpressExtraConfigContent: |     # Additional PHP configuration
  define('WP_DEBUG', true);
  define('WP_DEBUG_LOG', true);
  define('WP_MEMORY_LIMIT', '256M');

# Service configuration for external access
service:
  type: NodePort                    # Service type for external access
  ports:
    http: 80                       # Internal HTTP port
    https: 443                     # Internal HTTPS port
  nodePorts:
    http: "30004"                  # External HTTP port
    https: "30005"                 # External HTTPS port
  sessionAffinity: None            # Session affinity (None, ClientIP)

# Ingress configuration (alternative to NodePort)
ingress:
  enabled: false                   # Enable/disable ingress
  hostname: wordpress.homelab.local
  path: /
  pathType: Prefix
  tls: false                       # Enable HTTPS termination
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /

# Resource allocation for WordPress pods
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Pod configuration
replicaCount: 1                    # Number of WordPress pods
podSecurityContext:
  enabled: true
  fsGroup: 1001                    # Group for file system permissions

containerSecurityContext:
  enabled: true
  runAsUser: 1001                  # User ID for running WordPress
  runAsNonRoot: true
  readOnlyRootFilesystem: false

# Persistence configuration for WordPress files
persistence:
  enabled: false                   # Disable for learning (enables for production)
  storageClass: ""
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  dataSource: {}

# WordPress volume mounts for custom content
extraVolumes: []
extraVolumeMounts: []

# MariaDB database configuration
mariadb:
  enabled: true                    # Use bundled MariaDB (set false to use external DB)
  
  # MariaDB authentication
  auth:
    rootPassword: RootPassword123!
    database: wordpress            # WordPress database name
    username: wpuser              # WordPress database user
    password: WpUserPassword123!
  
  # MariaDB resource allocation
  primary:
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    
    # MariaDB persistence (disabled for learning)
    persistence:
      enabled: false
      storageClass: ""
      accessModes:
        - ReadWriteOnce
      size: 8Gi

# External database configuration (if mariadb.enabled=false)
externalDatabase:
  host: ""                         # External database host
  port: 3306                       # Database port
  user: wordpress                  # Database username
  password: ""                     # Database password
  database: wordpress              # Database name
  existingSecret: ""               # Existing secret with database credentials

# Health checks and probes
livenessProbe:
  enabled: true
  initialDelaySeconds: 120         # Wait 2 minutes before first check
  periodSeconds: 10                # Check every 10 seconds
  timeoutSeconds: 5                # Timeout after 5 seconds
  failureThreshold: 6              # Fail after 6 consecutive failures
  successThreshold: 1

readinessProbe:
  enabled: true
  initialDelaySeconds: 30          # Wait 30 seconds before first check
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6
  successThreshold: 1

# WordPress plugins and themes (advanced)
customPostInitScripts:
  install-plugins.sh: |
    #!/bin/bash
    # Install common WordPress plugins
    wp plugin install contact-form-7 --allow-root
    wp plugin install yoast-seo --allow-root
    wp plugin activate contact-form-7 --allow-root

# Monitoring and metrics
metrics:
  enabled: false                   # Enable WordPress metrics exporter
  serviceMonitor:
    enabled: false                 # Prometheus ServiceMonitor

# Network policies for security
networkPolicy:
  enabled: false                   # Enable network policies
  allowExternal: true              # Allow external traffic

# Additional labels and annotations
commonLabels:
  environment: learning
  app-type: cms

commonAnnotations:
  deployment-method: helm
  managed-by: kubernetes
```

**Learning Objectives:**
- ✅ Understand Helm charts and templating
- ✅ Learn to create custom charts
- ✅ Practice using public chart repositories
- ✅ Master Helm release management

---

## Phase 8: GitOps with ArgoCD

## Phase 8: GitOps with ArgoCD

### Step 17: Install ArgoCD

**Why We Create This:** Learn GitOps principles by installing ArgoCD, a declarative continuous delivery tool for Kubernetes. ArgoCD enables you to manage applications by storing their desired state in Git repositories and automatically synchronizing them to your cluster. This approach provides version control, rollback capabilities, and ensures that your cluster state matches what's defined in Git - a fundamental practice for production Kubernetes environments.

**Deploy ArgoCD:**

**Create:** `18-argocd-install.yaml`
```yaml
# ArgoCD Namespace - dedicated namespace for ArgoCD components
apiVersion: v1
kind: Namespace
metadata:
  name: argocd                     # Dedicated namespace for GitOps controller
  labels:
    name: argocd                   # Label for easy identification
---
# ServiceAccount for ArgoCD Server component
# ArgoCD Server serves the Web UI and API, needs permissions to manage applications
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-server              # Service account for the ArgoCD web server
  namespace: argocd
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd
---
# ClusterRole for ArgoCD Server - needs cluster-wide permissions to manage applications
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-server
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd
rules:
# Permissions to read cluster information
- apiGroups: [""]
  resources: ["*"]                 # All core API resources
  verbs: ["get", "list", "watch"]
# Permissions to manage applications across all namespaces
- apiGroups: ["apps"]
  resources: ["*"]                 # All apps API resources (deployments, etc.)
  verbs: ["*"]                     # All operations
- apiGroups: ["argoproj.io"]
  resources: ["*"]                 # All ArgoCD custom resources
  verbs: ["*"]
---
# ClusterRoleBinding to grant ArgoCD Server the necessary permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-server
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-server              # Reference to the ClusterRole above
subjects:
- kind: ServiceAccount
  name: argocd-server              # Reference to the ServiceAccount above
  namespace: argocd
---
# ArgoCD Server Deployment - Web UI and API server component
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
  namespace: argocd
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd
spec:
  replicas: 1                      # Single replica for learning environment
  selector:
    matchLabels:
      app.kubernetes.io/component: server
      app.kubernetes.io/name: argocd
  template:
    metadata:
      labels:
        app.kubernetes.io/component: server
        app.kubernetes.io/name: argocd
    spec:
      serviceAccountName: argocd-server
      containers:
      - name: argocd-server
        image: quay.io/argoproj/argocd:v2.8.4  # ArgoCD server image
        ports:
        - containerPort: 8080      # HTTP port for Web UI
          name: server
        - containerPort: 8083      # gRPC port for CLI and API access
          name: grpc
        command:
        - argocd-server            # Main ArgoCD server command
        args:
        - --insecure               # Disable TLS for learning (use TLS in production)
        - --staticassets           # Serve static assets for Web UI
        - /shared/app
        env:
        - name: ARGOCD_SERVER_INSECURE
          value: "true"            # Environment variable to disable TLS
        - name: ARGOCD_SERVER_ROOT_PATH
          value: "/"               # Root path for the server
        volumeMounts:
        - name: static-files       # Mount for static assets
          mountPath: /shared
        - name: tmp                # Temporary directory mount
          mountPath: /tmp
        resources:
          requests:
            memory: "256Mi"        # Minimum memory allocation
            cpu: "100m"            # Minimum CPU allocation
          limits:
            memory: "512Mi"        # Maximum memory allocation
            cpu: "500m"            # Maximum CPU allocation
        # Health checks to ensure server is running properly
        readinessProbe:
          httpGet:
            path: /healthz         # Health check endpoint
            port: 8080
          initialDelaySeconds: 10  # Wait 10 seconds before first check
          periodSeconds: 10        # Check every 10 seconds
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30  # Wait 30 seconds before first check
          periodSeconds: 30        # Check every 30 seconds
      volumes:
      - name: static-files         # Temporary storage for static assets
        emptyDir: {}
      - name: tmp                  # Temporary directory
        emptyDir: {}
---
# Service for ArgoCD Server - exposes Web UI and API
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd
spec:
  selector:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd
  ports:
  - name: http                     # HTTP port for Web UI access
    port: 80                       # Service port
    targetPort: 8080              # Container port
    nodePort: 30005               # External access port
  - name: grpc                     # gRPC port for CLI access
    port: 443                      # Service port
    targetPort: 8083              # Container port
    nodePort: 30006               # External access port for CLI
  type: NodePort                   # Expose service externally via NodePort
---
# ArgoCD Repository Server Deployment - handles Git repository operations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
  labels:
    app.kubernetes.io/component: repo-server
    app.kubernetes.io/name: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: repo-server
      app.kubernetes.io/name: argocd
  template:
    metadata:
      labels:
        app.kubernetes.io/component: repo-server
        app.kubernetes.io/name: argocd
    spec:
      containers:
      - name: argocd-repo-server
        image: quay.io/argoproj/argocd:v2.8.4
        ports:
        - containerPort: 8081      # Repository server port
          name: repo-server
        command:
        - argocd-repo-server       # Repository server command
        args:
        - --redis                  # Redis connection (using in-memory for simplicity)
        - argocd-redis:6379
        env:
        - name: ARGOCD_RECONCILIATION_TIMEOUT
          value: "180s"            # Timeout for Git operations
        - name: ARGOCD_REPO_SERVER_PARALLELISM_LIMIT
          value: "10"              # Maximum parallel Git operations
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: helm-working-dir   # Working directory for Helm operations
          mountPath: /helm-working-dir
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        # Health checks for repository server
        readinessProbe:
          tcpSocket:
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 30
      volumes:
      - name: tmp
        emptyDir: {}
      - name: helm-working-dir     # Temporary storage for Helm operations
        emptyDir: {}
---
# Service for Repository Server - internal communication only
apiVersion: v1
kind: Service
metadata:
  name: argocd-repo-server
  namespace: argocd
  labels:
    app.kubernetes.io/component: repo-server
    app.kubernetes.io/name: argocd
spec:
  selector:
    app.kubernetes.io/component: repo-server
    app.kubernetes.io/name: argocd
  ports:
  - name: repo-server
    port: 8081                     # Internal service port
    targetPort: 8081              # Container port
---
# Redis for ArgoCD - stores application state and cache
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-redis
  namespace: argocd
  labels:
    app.kubernetes.io/component: redis
    app.kubernetes.io/name: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: redis
      app.kubernetes.io/name: argocd
  template:
    metadata:
      labels:
        app.kubernetes.io/component: redis
        app.kubernetes.io/name: argocd
    spec:
      containers:
      - name: redis
        image: redis:7-alpine      # Lightweight Redis image
        ports:
        - containerPort: 6379      # Redis standard port
          name: redis
        args:
        - --save                   # Disable persistence for learning
        - ""
        - --appendonly             # Disable append-only file
        - "no"
        resources:
          requests:
            memory: "64Mi"
            cpu: "25m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        # Redis health checks
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 30
---
# Service for Redis - internal communication
apiVersion: v1
kind: Service
metadata:
  name: argocd-redis
  namespace: argocd
  labels:
    app.kubernetes.io/component: redis
    app.kubernetes.io/name: argocd
spec:
  selector:
    app.kubernetes.io/component: redis
    app.kubernetes.io/name: argocd
  ports:
  - name: redis
    port: 6379                     # Redis service port
    targetPort: 6379              # Container port
---
# ArgoCD Application Controller - manages application lifecycle
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-application-controller
  namespace: argocd
  labels:
    app.kubernetes.io/component: application-controller
    app.kubernetes.io/name: argocd
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: application-controller
      app.kubernetes.io/name: argocd
  template:
    metadata:
      labels:
        app.kubernetes.io/component: application-controller
        app.kubernetes.io/name: argocd
    spec:
      serviceAccountName: argocd-server  # Reuse server service account for simplicity
      containers:
      - name: argocd-application-controller
        image: quay.io/argoproj/argocd:v2.8.4
        command:
        - argocd-application-controller
        args:
        - --status-processors        # Number of concurrent status processors
        - "20"
        - --operation-processors     # Number of concurrent operation processors
        - "10"
        - --app-resync               # Application resync interval
        - "180"
        - --repo-server              # Repository server address
        - argocd-repo-server:8081
        - --redis                    # Redis connection
        - argocd-redis:6379
        env:
        - name: ARGOCD_CONTROLLER_REPLICAS
          value: "1"                 # Number of controller replicas
        - name: ARGOCD_RECONCILIATION_TIMEOUT
          value: "180s"              # Reconciliation timeout
        ports:
        - containerPort: 8082        # Controller metrics port
          name: controller
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        # Health checks for application controller
        readinessProbe:
          tcpSocket:
            port: 8082
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 8082
          initialDelaySeconds: 60
          periodSeconds: 30
---
# Secret containing initial admin password for ArgoCD
apiVersion: v1
kind: Secret
metadata:
  name: argocd-initial-admin-secret
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-secret
    app.kubernetes.io/part-of: argocd
type: Opaque
data:
  # Initial admin password: admin123 (base64 encoded)
  # In production, use a strong password and change it after first login
  password: YWRtaW4xMjM=
---
# ConfigMap for ArgoCD configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # Application configuration
  application.instanceLabelKey: argocd.argoproj.io/instance
  # Server configuration
  server.insecure: "true"          # Disable TLS for learning environment
  # Repository credentials template (for private repos)
  repositories: |
    - type: git
      url: https://github.com
      usernameSecret:
        name: github-secret
        key: username
      passwordSecret:
        name: github-secret
        key: password
```

**Install ArgoCD:**
```powershell
# Apply ArgoCD installation manifest
# This creates all ArgoCD components: server, repo-server, controller, and Redis
kubectl apply -f 18-argocd-install.yaml

# Wait for ArgoCD namespace to be created
kubectl wait --for=condition=ready --timeout=30s namespace/argocd

# Monitor the deployment progress
# All ArgoCD components must be ready before proceeding
kubectl get pods -n argocd -w

# Wait for ArgoCD server to be ready (may take 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Wait for other components to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-repo-server -n argocd
kubectl wait --for=condition=available --timeout=300s deployment/argocd-application-controller -n argocd
kubectl wait --for=condition=available --timeout=300s deployment/argocd-redis -n argocd

# Verify all pods are running
kubectl get pods -n argocd

# Check services are accessible
kubectl get services -n argocd

# Verify ArgoCD server is responding
kubectl port-forward -n argocd service/argocd-server 8080:80 &

# Test ArgoCD API endpoint
curl -k http://localhost:8080/healthz

# Stop port-forward
pkill -f "port-forward.*argocd-server"

# Access ArgoCD UI through NodePort
echo "ArgoCD Web UI: http://localhost:30005"
echo "Username: admin"
echo "Password: admin123"
echo ""
echo "ArgoCD CLI Access (if you install ArgoCD CLI):"
echo "argocd login localhost:30006 --username admin --password admin123 --insecure"

# Optional: Install ArgoCD CLI for command-line management
# Download from: https://github.com/argoproj/argo-cd/releases/latest
# For Windows: Download argocd-windows-amd64.exe and rename to argocd.exe

# Verify ArgoCD installation
kubectl get all -n argocd

# Check ArgoCD logs if there are issues
kubectl logs -n argocd deployment/argocd-server -f
kubectl logs -n argocd deployment/argocd-application-controller -f
kubectl logs -n argocd deployment/argocd-repo-server -f
```

### Step 18: Prepare Git Repository Structure

**Why We Create This:** Organize Kubernetes manifests in a GitOps-friendly directory structure that ArgoCD can monitor and deploy. This demonstrates how to structure applications for GitOps workflows, where each application has its own directory containing all necessary Kubernetes manifests. This pattern enables ArgoCD to track changes and automatically synchronize the desired state from Git to your cluster.

**Create Application Manifests Directory Structure:**

```powershell
# Create directory structure for GitOps applications
# Each application gets its own directory containing all related manifests
mkdir -p gitops-apps/nginx-app
mkdir -p gitops-apps/redis-app
mkdir -p gitops-apps/multi-tier-app

# Verify directory structure
tree gitops-apps/
# gitops-apps/
# ├── nginx-app/
# ├── redis-app/
# └── multi-tier-app/
```

**Create:** `gitops-apps/nginx-app/deployment.yaml`
```yaml
# GitOps-managed Nginx application deployment
# This file represents the desired state for our web application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-nginx                 # Unique name for GitOps-managed nginx
  namespace: default
  labels:
    app: gitops-nginx
    managed-by: argocd               # Label indicating GitOps management
    app.kubernetes.io/name: nginx
    app.kubernetes.io/component: web-server
    app.kubernetes.io/part-of: gitops-demo
spec:
  replicas: 2                        # Start with 2 replicas (can be modified later)
  selector:
    matchLabels:
      app: gitops-nginx
  template:
    metadata:
      labels:
        app: gitops-nginx
        version: "1.21"              # Version label for tracking
    spec:
      containers:
      - name: nginx
        image: nginx:1.21            # Specific version for predictable deployments
        ports:
        - containerPort: 80
          name: http
        # Resource limits for predictable performance
        resources:
          requests:
            memory: "64Mi"           # Minimum memory allocation
            cpu: "50m"               # Minimum CPU allocation
          limits:
            memory: "128Mi"          # Maximum memory allocation
            cpu: "100m"              # Maximum CPU allocation
        # Health checks to ensure container is ready
        readinessProbe:
          httpGet:
            path: /                  # Check root path
            port: 80
          initialDelaySeconds: 5     # Wait 5 seconds before first check
          periodSeconds: 10          # Check every 10 seconds
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15    # Wait 15 seconds before first check
          periodSeconds: 30          # Check every 30 seconds
        # Volume mount for custom configuration (to be added later)
        volumeMounts:
        - name: nginx-config
          mountPath: /usr/share/nginx/html
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-html           # Reference to ConfigMap (created below)
          defaultMode: 0644          # File permissions
---
# Service to expose the GitOps-managed Nginx application
apiVersion: v1
kind: Service
metadata:
  name: gitops-nginx-service
  namespace: default
  labels:
    app: gitops-nginx
    managed-by: argocd
spec:
  selector:
    app: gitops-nginx              # Select pods with this label
  ports:
  - name: http
    port: 80                       # Service port
    targetPort: 80                 # Container port
    nodePort: 30007               # External access port
  type: NodePort                   # Expose externally for testing
```

**Create:** `gitops-apps/nginx-app/configmap.yaml`
```yaml
# ConfigMap containing custom HTML content for the nginx application
# This demonstrates how configuration is managed separately from the application code
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
  namespace: default
  labels:
    app: gitops-nginx
    managed-by: argocd
data:
  # Custom HTML content that can be updated independently
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>GitOps Demo with ArgoCD</title>
        <style>
            body { 
                font-family: Arial, sans-serif; 
                background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                color: white;
                text-align: center;
                padding: 50px;
            }
            .container {
                background: rgba(255,255,255,0.1);
                padding: 30px;
                border-radius: 10px;
                display: inline-block;
            }
            .version { 
                background: #28a745; 
                padding: 5px 15px; 
                border-radius: 20px; 
                display: inline-block;
                margin: 10px 0;
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🚀 Hello from GitOps!</h1>
            <div class="version">Version: 1.0.0</div>
            <p>This application is managed by ArgoCD</p>
            <p>Deployed from Git repository</p>
            <p>Pod Name: <code id="hostname">Loading...</code></p>
            <p>Deployment managed declaratively via GitOps principles</p>
            
            <h3>GitOps Benefits:</h3>
            <ul style="text-align: left; display: inline-block;">
                <li>🔄 Automatic synchronization from Git</li>
                <li>📝 Declarative configuration management</li>
                <li>🔍 Version control and audit trail</li>
                <li>🔄 Easy rollbacks and updates</li>
                <li>👥 Collaboration through Pull Requests</li>
            </ul>
        </div>
        
        <script>
            // Display the hostname (pod name) if available
            fetch('/hostname').then(r => r.text()).then(hostname => {
                document.getElementById('hostname').textContent = hostname;
            }).catch(() => {
                document.getElementById('hostname').textContent = window.location.hostname;
            });
        </script>
    </body>
    </html>
  
  # Additional configuration files can be added here
  nginx.conf: |
    # Custom nginx configuration
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ =404;
        }
        
        # Endpoint to return hostname (pod name)
        location /hostname {
            return 200 $hostname;
            add_header Content-Type text/plain;
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # Basic security headers
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;
        add_header X-XSS-Protection "1; mode=block";
    }
```

**Create:** `gitops-apps/redis-app/deployment.yaml`
```yaml
# GitOps-managed Redis deployment for caching and session storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-redis
  namespace: default
  labels:
    app: gitops-redis
    managed-by: argocd
    app.kubernetes.io/name: redis
    app.kubernetes.io/component: cache
    app.kubernetes.io/part-of: gitops-demo
spec:
  replicas: 1                        # Redis typically runs as single instance
  selector:
    matchLabels:
      app: gitops-redis
  template:
    metadata:
      labels:
        app: gitops-redis
        version: "alpine"
    spec:
      containers:
      - name: redis
        image: redis:7-alpine          # Lightweight Alpine-based Redis
        ports:
        - containerPort: 6379
          name: redis
        command:
        - redis-server                 # Redis server command
        args:
        - /etc/redis/redis.conf        # Use custom configuration
        # Resource allocation for Redis
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        # Health checks for Redis
        readinessProbe:
          exec:
            command:
            - redis-cli               # Use Redis CLI for health check
            - ping
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 30
        # Volume mounts for configuration and data
        volumeMounts:
        - name: redis-config
          mountPath: /etc/redis
          readOnly: true
        - name: redis-data             # Temporary data directory
          mountPath: /data
      volumes:
      - name: redis-config
        configMap:
          name: redis-config           # Reference to Redis configuration
          defaultMode: 0644
      - name: redis-data
        emptyDir: {}                   # Temporary storage (use PVC for persistence)
---
# Service for Redis - internal access only
apiVersion: v1
kind: Service
metadata:
  name: gitops-redis-service
  namespace: default
  labels:
    app: gitops-redis
    managed-by: argocd
spec:
  selector:
    app: gitops-redis
  ports:
  - name: redis
    port: 6379                       # Standard Redis port
    targetPort: 6379
  type: ClusterIP                    # Internal service only
```

**Create:** `gitops-apps/redis-app/configmap.yaml`
```yaml
# Redis configuration managed via ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: default
  labels:
    app: gitops-redis
    managed-by: argocd
data:
  # Redis server configuration
  redis.conf: |
    # Redis configuration for GitOps demo
    # Network and security settings
    bind 0.0.0.0                     # Listen on all interfaces
    protected-mode no                # Disable protected mode for internal use
    port 6379                        # Standard Redis port
    
    # Memory management
    maxmemory 100mb                  # Maximum memory usage
    maxmemory-policy allkeys-lru     # Eviction policy when memory full
    
    # Persistence settings (disabled for demo)
    save ""                          # Disable automatic snapshots
    appendonly no                    # Disable append-only file
    
    # Logging
    loglevel notice                  # Log level
    logfile ""                       # Log to stdout
    
    # Performance tuning
    tcp-keepalive 300                # TCP keepalive
    timeout 0                        # Client timeout (0 = no timeout)
    tcp-backlog 511                  # TCP listen backlog
    
    # Database settings
    databases 16                     # Number of databases
    
    # Security (basic settings for demo)
    # requirepass mypassword         # Uncomment to require password
    
    # Lua scripting
    lua-time-limit 5000             # Lua script execution time limit
    
    # Slow log (for monitoring)
    slowlog-log-slower-than 10000   # Log queries slower than 10ms
    slowlog-max-len 128             # Maximum slow log entries
    
    # Client output buffer limits
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit replica 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
```

**Create:** `gitops-apps/multi-tier-app/frontend-deployment.yaml`
```yaml
# Multi-tier application - Frontend component
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: default
  labels:
    app: frontend-app
    tier: frontend
    managed-by: argocd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-app
      tier: frontend
  template:
    metadata:
      labels:
        app: frontend-app
        tier: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "25m"
          limits:
            memory: "64Mi"
            cpu: "50m"
        volumeMounts:
        - name: frontend-config
          mountPath: /usr/share/nginx/html
      volumes:
      - name: frontend-config
        configMap:
          name: frontend-html
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: default
  labels:
    app: frontend-app
    tier: frontend
spec:
  selector:
    app: frontend-app
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30008
  type: NodePort
```

**Test the GitOps Structure:**
```powershell
# Verify directory structure is created correctly
Get-ChildItem -Recurse gitops-apps/

# Validate YAML syntax for all manifests
kubectl apply --dry-run=client -f gitops-apps/nginx-app/
kubectl apply --dry-run=client -f gitops-apps/redis-app/
kubectl apply --dry-run=client -f gitops-apps/multi-tier-app/

# Initialize git repository (if not already done)
git init
git add gitops-apps/
git commit -m "Initial GitOps application structure"

# Note: In production, you would push this to a Git repository
# that ArgoCD can access (GitHub, GitLab, etc.)
```

### Step 19: Create ArgoCD Applications

**Why We Create This:** Define ArgoCD Application resources that tell ArgoCD which Git repositories to monitor and how to deploy applications. These Application manifests are the core of GitOps - they describe the desired state, source repository, and deployment target for each application. ArgoCD continuously monitors these definitions and ensures the cluster state matches what's defined in Git.

**Create:** `19-argocd-nginx-app.yaml`
```yaml
# ArgoCD Application definition for the Nginx web application
# This tells ArgoCD to monitor the nginx-app directory and deploy it to the cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app                   # Name of the application in ArgoCD
  namespace: argocd                 # ArgoCD applications must be in argocd namespace
  labels:
    app.kubernetes.io/name: nginx-gitops
    managed-by: argocd
  # Finalizers ensure proper cleanup when application is deleted
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  # Project determines RBAC and resource permissions
  project: default                  # Use default project (can create custom projects)
  
  # Source configuration - where ArgoCD gets the manifests
  source:
    # Repository URL - in production, use HTTPS Git URLs like:
    # repoURL: 'https://github.com/username/repo.git'
    repoURL: 'https://github.com/your-username/homelab-k8s.git'
    targetRevision: main            # Git branch, tag, or commit to track
    path: gitops-apps/nginx-app     # Directory containing the manifests
    
    # Optional: Directory-specific configuration
    directory:
      recurse: true                 # Include subdirectories
      jsonnet: {}                   # Enable Jsonnet support if needed
  
  # Destination configuration - where ArgoCD deploys the application
  destination:
    server: 'https://kubernetes.default.svc'  # Target Kubernetes cluster
    namespace: default              # Target namespace for deployment
  
  # Sync policy - how ArgoCD manages the application lifecycle
  syncPolicy:
    # Automated sync configuration
    automated:
      prune: true                   # Remove resources not in Git
      selfHeal: true               # Fix drift automatically
      allowEmpty: false            # Don't sync if no resources found
    
    # Sync options for fine-grained control
    syncOptions:
    - CreateNamespace=true          # Create namespace if it doesn't exist
    - PrunePropagationPolicy=foreground  # How to handle resource deletion
    - PruneLast=true               # Prune resources after applying new ones
    
    # Retry configuration for failed syncs
    retry:
      limit: 5                     # Maximum retry attempts
      backoff:
        duration: 5s               # Initial retry delay
        factor: 2                  # Backoff multiplier
        maxDuration: 3m            # Maximum retry delay
  
  # Health check configuration
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas               # Ignore replica differences (for HPA)
  
  # Revision history limit
  revisionHistoryLimit: 10         # Keep last 10 revisions for rollback
```

**Create:** `19-argocd-redis-app.yaml`
```yaml
# ArgoCD Application definition for Redis caching service
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-app
  namespace: argocd
  labels:
    app.kubernetes.io/name: redis-gitops
    managed-by: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: 'https://github.com/your-username/homelab-k8s.git'
    targetRevision: main
    path: gitops-apps/redis-app
    
    # Redis-specific source configuration
    directory:
      recurse: true
  
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    
    syncOptions:
    - CreateNamespace=true
    - RespectIgnoreDifferences=true
    
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 1m
  
  # Health assessment for Redis
  ignoreDifferences:
  - group: ""
    kind: ConfigMap
    jsonPointers:
    - /data/redis.conf             # Ignore config differences that don't affect operation
```

**Create:** `19-argocd-multi-tier-app.yaml`
```yaml
# ArgoCD Application for multi-tier application demo
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-tier-app
  namespace: argocd
  labels:
    app.kubernetes.io/name: multi-tier-gitops
    managed-by: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: 'https://github.com/your-username/homelab-k8s.git'
    targetRevision: main
    path: gitops-apps/multi-tier-app
  
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  
  syncPolicy:
    # Manual sync for demonstration purposes
    # automated: {}                # Uncomment for automatic sync
    
    syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true      # Only apply resources that are out of sync
    
  # Application health check configuration
  ignoreDifferences: []
  
  # Information links for documentation
  info:
  - name: 'Documentation'
    value: 'https://github.com/your-username/homelab-k8s/blob/main/README.md'
  - name: 'Health Dashboard'
    value: 'http://localhost:30008'
```

**Create ArgoCD Project (Optional - Advanced):**

**Create:** `19-argocd-project.yaml`
```yaml
# Custom ArgoCD Project for better organization and security
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: homelab-project
  namespace: argocd
  labels:
    managed-by: argocd
spec:
  # Description of the project
  description: 'Homelab Learning Project for GitOps demonstrations'
  
  # Source repositories that this project can deploy from
  sourceRepos:
  - 'https://github.com/your-username/homelab-k8s.git'
  - 'https://github.com/your-username/*'  # Allow all repos from user
  - '*'                           # Allow all repositories (for learning)
  
  # Destination clusters and namespaces
  destinations:
  - namespace: 'default'
    server: 'https://kubernetes.default.svc'
  - namespace: 'learning-*'       # Allow namespaces starting with learning-
    server: 'https://kubernetes.default.svc'
  
  # RBAC configuration for the project
  roles:
  - name: admin                   # Admin role for project
    description: 'Full access to homelab project applications'
    policies:
    - p, proj:homelab-project:admin, applications, *, homelab-project/*, allow
    - p, proj:homelab-project:admin, repositories, *, *, allow
    groups:
    - homelab-admins
  
  - name: developer               # Developer role with limited access
    description: 'Developer access for homelab project'
    policies:
    - p, proj:homelab-project:developer, applications, get, homelab-project/*, allow
    - p, proj:homelab-project:developer, applications, sync, homelab-project/*, allow
    groups:
    - homelab-developers
  
  # Cluster resource whitelist - what can be deployed
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRole
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRoleBinding
  
  # Namespace resource whitelist
  namespaceResourceWhitelist:
  - group: ''
    kind: '*'                     # Allow all core resources
  - group: 'apps'
    kind: '*'                     # Allow all apps resources
  - group: 'extensions'
    kind: '*'                     # Allow extensions
  
  # Orphaned resources monitoring
  orphanedResources:
    warn: true                    # Warn about orphaned resources
    ignore:
    - group: ''
      kind: Secret
      name: 'argocd-*'           # Ignore ArgoCD secrets
```

**Deploy ArgoCD Applications:**
```powershell
# First, update the repoURL in the application manifests to point to your actual Git repository
# For local testing, you can use file:// URLs or create a simple Git server

# Option 1: Use local file system (for testing only)
# Update repoURL in all application YAMLs to:
# repoURL: 'file:///c/Users/sinadvd/Documents/VScode/homelab-prj/k8s-homelab/homelab'

# Option 2: Use a Git repository (recommended)
# 1. Create a Git repository (GitHub, GitLab, etc.)
# 2. Push your gitops-apps directory to the repository
# 3. Update repoURL to your repository URL

# Deploy the ArgoCD applications
kubectl apply -f 19-argocd-nginx-app.yaml
kubectl apply -f 19-argocd-redis-app.yaml
kubectl apply -f 19-argocd-multi-tier-app.yaml

# Optional: Deploy custom project
kubectl apply -f 19-argocd-project.yaml

# Check ArgoCD applications status
kubectl get applications -n argocd

# Get detailed information about applications
kubectl describe application nginx-app -n argocd
kubectl describe application redis-app -n argocd

# Check application health and sync status
kubectl get applications -n argocd -o wide

# Watch applications sync (this may take a few minutes)
watch kubectl get applications -n argocd

# Verify deployed resources in target namespace
kubectl get all -l managed-by=argocd

# Check specific application deployments
kubectl get pods -l app=gitops-nginx
kubectl get pods -l app=gitops-redis

# Test application accessibility
echo "Nginx GitOps App: http://localhost:30007"
echo "Redis is internal service - test via port-forward if needed"

# Port-forward to test Redis if needed
kubectl port-forward service/gitops-redis-service 6379:6379 &
redis-cli -h localhost ping
pkill -f "port-forward.*gitops-redis"
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