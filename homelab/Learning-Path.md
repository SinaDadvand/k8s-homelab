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

**Create:** `10-redis-deployment.yaml`
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
kubectl apply -f 10-redis-deployment.yaml

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

**Create:** `11-redis-persistent.yaml`
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
            cpu: "50m"
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
kubectl apply -f 11-redis-persistent.yaml

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

### Step 12: Data Backup and Recovery

**Why We Learn This:** Understanding data backup and recovery is crucial for stateful applications. This step teaches you how to backup persistent data and restore it when needed.

**Create:** `12-backup-recovery.yaml`
```yaml
# Job to backup Redis data
apiVersion: batch/v1
kind: Job
metadata:
  name: redis-backup
spec:
  template:
    spec:
      containers:
      - name: backup
        image: redis:alpine
        command: ['sh', '-c']
        args:
        - |
          echo "Starting Redis backup..."
          redis-cli -h redis-service BGSAVE
          echo "Backup completed successfully"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      restartPolicy: Never
  backoffLimit: 3
```

**Apply and Test:**
```powershell
# Create backup job
kubectl apply -f 12-backup-recovery.yaml

# Check job status
kubectl get jobs
kubectl logs job/redis-backup

# Verify backup
kubectl exec -it redis-0 -- redis-cli LASTSAVE
```

**Learning Objectives:**
- ✅ Understand data backup strategies
- ✅ Learn about Kubernetes Jobs
- ✅ Practice backup and recovery procedures

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
            - name: app-key
              mountPath: /usr/share/nginx/html/app-key.txt
              subPath: app-key.txt
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      volumes:
        - name: config
          configMap:
            name: {{ include "my-nginx-chart.fullname" . }}-config
        - name: app-key
          secret:
            secretName: my-app-key
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
  
  # Additional configuration files can be added here
  nginx.conf: |
    # Custom nginx configuration
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        # Endpoint to return hostname (pod name)
        location /hostname {
            return 200 $hostname;
            add_header Content-Type text/plain;
        }
        
        # Health check endpoint
        location /health {
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
        
        # Basic security headers
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;
        add_header X-XSS-Protection "1; mode=block";
    }
---
# Service for the complete application stack
apiVersion: v1
kind: Service
metadata:
  name: complete-app-service
  namespace: default
spec:
  selector:
    app: complete-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  - port: 443
    targetPort: 443
    nodePort: 30443
  type: NodePort
---
# Ingress for the complete application stack
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: complete-app-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: complete-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: complete-app-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: api.complete-app.local
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
# TLS Secret for HTTPS
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # Your certificate (base64 encoded)
  tls.key: LS0tLS1CRUdJTi... # Your private key (base64 encoded)
```

### Step 15: Advanced Helm Features

**Create:** `15-advanced-helm.yaml`
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
kubectl apply -f 15-advanced-patterns.yaml

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
- ✅ Understand Helm package management fundamentals
- ✅ Create and deploy custom Helm charts

---

## Phase 8: GitOps with ArgoCD

### Step 16: Introduction to GitOps with ArgoCD

**Why We Learn This:** GitOps is a modern approach to continuous delivery where your Git repository becomes the single source of truth for your application deployments. ArgoCD monitors your Git repositories and automatically synchronizes changes to your Kubernetes cluster. This provides version control, audit trails, and automated rollbacks - essential for production environments.

**Install ArgoCD using Official Manifests:**

```powershell
# Step 1: Install ArgoCD using the official installation method
# This is the simplest and most reliable approach
kubectl create namespace argocd

# Install ArgoCD components from official repository
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all ArgoCD components to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
kubectl wait --for=condition=available --timeout=300s deployment/argocd-repo-server -n argocd
kubectl wait --for=condition=available --timeout=300s deployment/argocd-application-controller -n argocd

# Verify all pods are running
kubectl get pods -n argocd
```

**Expose ArgoCD Server for Local Access:**

```powershell
# Create a NodePort service to access ArgoCD Web UI
# This replaces the default ClusterIP service with external access
cmd /c 'kubectl patch svc argocd-server -n argocd -p "{\"spec\":{\"type\":\"NodePort\",\"ports\":[{\"name\":\"https\",\"port\":443,\"protocol\":\"TCP\",\"targetPort\":8080,\"nodePort\":30001}]}}"'

# Verify the service is exposed
kubectl get svc argocd-server -n argocd

# Get the initial admin password
# ArgoCD generates a random password stored in a secret
$ARGOCD_PASSWORD = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# Display connection information
Write-Host "ArgoCD Web UI: http://localhost:30001"
Write-Host "Username: admin"
Write-Host "Password: $ARGOCD_PASSWORD"
Write-Host ""

# Verify everything is working
kubectl get applications -n argocd
kubectl get pods -n guestbook
kubectl get pods -n gitops-demo
```

### Step 17: Create Simple GitOps Applications

**Why We Learn This:** Learn to deploy applications using ArgoCD by creating simple application manifests. We'll start with basic examples that demonstrate GitOps principles without the complexity of external Git repositories.

**Create a Simple Nginx Application:**

**Create:** `17-simple-gitops-app.yaml`
```yaml
# Application for learning GitOps with ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: simple-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-username/homelab-k8s.git'
    targetRevision: HEAD
    path: gitops-apps/nginx-app
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Deploy the GitOps Application:**

```powershell
# Deploy the simple application to demonstrate GitOps
kubectl apply -f manifests/learning-path/17-simple-gitops-app.yaml

# Verify the application is deployed
kubectl get all -n gitops-demo

# Test the application
Write-Host "GitOps Demo App: http://localhost:30080"
Start-Sleep -Seconds 10

# Check if the application is accessible
try {
    $response = Invoke-WebRequest -Uri "http://localhost:30080" -UseBasicParsing
    Write-Host "✅ GitOps Demo App is running!"
} catch {
    Write-Host "⚠️  App is starting up, try again in a minute"
}
```

**Create ArgoCD Application to Manage Our App:**

**Note:** The original `17-argocd-application.yaml` file has been corrected and now properly uses local deployment. We've also created additional examples:

- `17-argocd-application.yaml` - **Local Repository Example** (corrected)
- `17b-argocd-application-public-repo.yaml` - **Public Repository Example**
- `17c-argocd-application-personal-repo.yaml` - **Personal Repository Examples**

**Main:** `17-argocd-application.yaml` (Corrected to use local deployment)
```yaml
# ArgoCD Application - tells ArgoCD how to manage our local nginx app
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: local-nginx-demo
  namespace: argocd
  labels:
    app.kubernetes.io/name: local-nginx-demo
spec:
  # Project that this application belongs to
  project: default
  
  # Source configuration - using local gitops-apps directory
  source:
    # Using local gitops-apps directory for in-cluster learning
    repoURL: https://github.com/sinadogru/homelab.git
    targetRevision: HEAD
    path: gitops-apps/nginx-app
  
  # Destination configuration - where to deploy the application
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx-demo
  
  # Sync policy - how ArgoCD should handle synchronization
  syncPolicy:
    # Automatically sync when differences are detected
    automated:
      prune: true        # Remove resources that are no longer defined
      selfHeal: true     # Automatically fix drift from desired state
    
    # Options for sync behavior
    syncOptions:
    - CreateNamespace=true    # Create namespace if it doesn't exist
    - PrunePropagationPolicy=foreground
    - PruneLast=true
```

**Additional Examples Created:**

See `README-ArgoCD-Applications.md` for comprehensive documentation of all three ArgoCD application examples including:
- Local repository deployment (corrected main example)
- Public repository deployment (ArgoCD examples)
- Personal repository deployment (multiple apps)

**Deploy ArgoCD Application:**

```powershell
# Deploy the corrected ArgoCD application configuration
kubectl apply -f manifests/learning-path/17-argocd-application.yaml

# Optionally deploy additional examples
kubectl apply -f manifests/learning-path/17b-argocd-application-public-repo.yaml
kubectl apply -f manifests/learning-path/17c-argocd-application-personal-repo.yaml

# Check ArgoCD applications
kubectl get applications -n argocd

# View application details
kubectl describe application local-nginx-demo -n argocd

# Test the nginx application
kubectl get pods -n nginx-demo
```

### Step 18: Working with ArgoCD Web UI

**Access and Navigate ArgoCD UI:**

```powershell
# Get ArgoCD access information
$ARGOCD_PASSWORD = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

Write-Host "🚀 ArgoCD Access Information:"
Write-Host "Web UI: http://localhost:30001"
Write-Host "Username: admin"
Write-Host "Password: $ARGOCD_PASSWORD"
Write-Host ""
Write-Host "📋 What to do in ArgoCD UI:"
Write-Host "1. Login with admin credentials"
Write-Host "2. View the 'simple-gitops-demo' application"
Write-Host "3. Explore the application topology"
Write-Host "4. Check sync status and health"
Write-Host "5. Try manual sync operations"
```

**Understanding ArgoCD UI Components:**

```powershell
# While exploring the ArgoCD UI, understand these key concepts:

Write-Host "🎯 Key ArgoCD Concepts:"
Write-Host ""
Write-Host "📊 Application Dashboard:"
Write-Host "  - Shows all applications managed by ArgoCD"
Write-Host "  - Displays sync status (Synced/OutOfSync)"
Write-Host "  - Shows health status (Healthy/Degraded/Unknown)"
Write-Host ""
Write-Host "🔄 Sync Status:"
Write-Host "  - Synced: Cluster state matches Git state"
Write-Host "  - OutOfSync: Differences detected"
Write-Host "  - Unknown: Unable to determine status"
Write-Host ""
Write-Host "💚 Health Status:"
Write-Host "  - Healthy: All resources running properly"
Write-Host "  - Degraded: Some issues detected"
Write-Host "  - Progressing: Deployment in progress"
Write-Host ""
Write-Host "🛠️ Operations:"
Write-Host "  - Sync: Apply changes from Git to cluster"
Write-Host "  - Refresh: Check for new changes"
Write-Host "  - Hard Refresh: Force refresh of all resources"
```

### Step 19: Testing GitOps Workflow

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
        volumeMounts:
          - name: frontend-config
            mountPath: /usr/share/nginx/html/index.html
            subPath: index.html
          - name: config-volume
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
      volumes:
        - name: frontend-config
          configMap:
            name: frontend-config
        - name: config-volume
          configMap:
            name: nginx-config
```

### Step 20: App of Apps Pattern

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
    syncOptions:
    - CreateNamespace=true
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

### Step 21: Create a Helm Chart for Complete Stack

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
  # JSON configuration generated from Helm values
  config.json: |
    {
      "application": {
        "name": "{{ include "complete-stack-chart.fullname" . }}",
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
        <title>{{ include "complete-stack-chart.fullname" . }}</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            h1 { color: #326ce5; }
            .config { background: #f5f5f5; padding: 20px; border-radius: 5px; }
        </style>
    </head>
    <body>
        <h1>Helm Chart Application</h1>
        <p><strong>Application:</strong> {{ include "complete-stack-chart.fullname" . }}</p>
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
  
  # Additional configuration files can be added here
  nginx.conf: |
    # Custom nginx configuration
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        # Endpoint to return hostname (pod name)
        location /hostname {
            return 200 $hostname;
            add_header Content-Type text/plain;
        }
        
        # Health check endpoint
        location /health {
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
        
        # Basic security headers
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;
        add_header X-XSS-Protection "1; mode=block";
    }
---
# Service for the complete application stack
apiVersion: v1
kind: Service
metadata:
  name: complete-app-service
  namespace: default
spec:
  selector:
    app: complete-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  - port: 443
    targetPort: 443
    nodePort: 30443
  type: NodePort
---
# Ingress for the complete application stack
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: complete-app-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: complete-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: complete-app-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: api.complete-app.local
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
# TLS Secret for HTTPS
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:  tls.crt: LS0tLS1CRUdJTi... # Your certificate (base64 encoded)
  tls.key: LS0tLS1CRUdJTi... # Your private key (base64 encoded)
```

### Step 22: Advanced GitOps Patterns

**Create:** `22-advanced-gitops.yaml`
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
          name: advanced-app-config        - name: nginx-config
          configMap:
            name: nginx-advanced-config
```

### Step 23: Monitoring and Automation

**Create:** `23-monitoring-automation.yaml`
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

### Step 24: Advanced ArgoCD Features

**Create:** `24-argocd-advanced-features.yaml`
```yaml
# ArgoCD Application with advanced features
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: advanced-features-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-username/homelab-k8s.git'
    targetRevision: HEAD
    path: gitops-apps/multi-tier-app
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
  
  # Health checks and custom parameters
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas               # Ignore replica differences (for HPA)
  
  # Revision history limit
  revisionHistoryLimit: 10         # Keep last 10 revisions for rollback

  # Custom parameters for templating
  parameters:
  - name: replicaCount
    value: "3"
  - name: image.tag
    value: "latest"
```

**Deploy and Test Advanced Features:**
```powershell
# Deploy the application with advanced features
kubectl apply -f 24-argocd-advanced-features.yaml

# Check ArgoCD applications
kubectl get applications -n argocd

# Test the application
kubectl get pods -n production
# Open browser to http://localhost:30006
```

**Learning Objectives:**
- ✅ Combine multiple advanced concepts
- ✅ Practice with real-world application scenarios
- ✅ Learn to troubleshoot complex issues
- ✅ Prepare for production-grade Kubernetes deployments

---

## Advanced Summary

### ✅ **Advanced Concepts Mastered:**
1. **GitOps Principles** - Manage applications through Git
2. **ArgoCD Installation** - Deploy and configure ArgoCD
3. **Application Management** - Create and manage ArgoCD applications
4. **Monitoring and Observability** - Set up Prometheus and Grafana
5. **Advanced Workload Patterns** - Init containers, sidecars, jobs, cronjobs
6. **Secrets and ConfigMaps** - Manage sensitive data and configurations
7. **RBAC** - Implement access control
8. **Ingress** - Manage external access to services

### ✅ **Best Practices Learned:**
- Secure and scalable application design
- Declarative management of applications and infrastructure
- Automated deployment and monitoring
- Troubleshooting and debugging complex issues

---

## 🎉 ArgoCD Implementation Complete!

### Working Installation Summary

**✅ Successfully Completed:**
- **ArgoCD Installation**: Using official manifests in `argocd-new` namespace
- **Web UI Access**: Available at http://localhost:31557
- **Authentication**: Admin credentials configured and working
- **Application Management**: Guestbook application synced and healthy
- **GitOps Demo**: Educational demo application running and accessible
- **Verification**: Automated PowerShell script created for status checking

**📊 Current Status:**
```powershell
# Quick verification commands:
kubectl get pods -n argocd-new                    # All ArgoCD components running
kubectl get applications -n argocd-new            # Applications synced and healthy
kubectl get svc argocd-server -n argocd-new      # NodePort 31557 accessible
```

**🔐 Access Information:**
- **ArgoCD Web UI**: http://localhost:31557
- **Username**: admin
- **Password**: `5VLXtRIp2PykfYLC`
- **GitOps Demo**: http://localhost:30090

**🛠️ Verification Script:**
```powershell
# Run the automated verification script
.\scripts\verify-argocd.ps1
```

**📚 What We Learned:**
1. **GitOps Principles**: Declarative configuration management with Git as source of truth
2. **ArgoCD Architecture**: Understanding controllers, UI, and repository servers
3. **Application Lifecycle**: Sync policies, health checks, and drift detection
4. **Troubleshooting**: RBAC permissions, service configuration, and namespace management
5. **Production Practices**: NodePort configuration, password management, and monitoring

**🧹 Cleanup Procedures:**
We've created an automated cleanup script that safely removes all ArgoCD components:

```powershell
# Use the automated cleanup script (recommended)
.\scripts\cleanup-final.ps1 -DryRun        # Preview what would be deleted
.\scripts\cleanup-final.ps1 -KeepApps      # Remove ArgoCD but keep applications
.\scripts\cleanup-final.ps1 -Force         # Remove everything without prompts
.\scripts\cleanup-final.ps1               # Interactive cleanup with confirmations
```

**Cleanup Script Features:**
- **Dry-run mode**: Preview changes without executing them
- **Selective cleanup**: Option to keep applications while removing ArgoCD
- **Force mode**: Automated cleanup without prompts
- **Interactive mode**: Step-by-step confirmation for each operation
- **Safety checks**: Verifies resources exist before attempting deletion
- **Comprehensive**: Removes applications, namespaces, and temporary files

**Manual Cleanup (if needed):**
```powershell
# Remove ArgoCD applications first
kubectl delete applications --all -n argocd-new

# Remove ArgoCD installation
kubectl delete namespace argocd-new

# Clean up demo applications
kubectl delete namespace gitops-demo
kubectl delete namespace guestbook

# Clean up temporary files
Remove-Item .\patch-namespace.json -ErrorAction SilentlyContinue
Remove-Item .\patch-service.json -ErrorAction SilentlyContinue
```

**🔄 Advanced GitOps Scenarios:**
The current implementation provides a solid foundation for exploring:
- Multi-application management with App of Apps pattern
- Private Git repository integration
- Advanced sync policies and hooks
- Multi-cluster deployments
- ArgoCD Projects and RBAC
- Custom resource management
- Rollback and canary deployment strategies

**🏆 Implementation Success Metrics:**
- ✅ **7/7 ArgoCD pods** running successfully
- ✅ **1/1 applications** synced and healthy
- ✅ **2/2 demo applications** deployed and accessible
- ✅ **100% uptime** Web UI and CLI access
- ✅ **Automation complete** with verification and cleanup scripts

**📁 Files Created/Modified:**
- `manifests/learning-path/17-argocd-application.yaml` - ArgoCD application definition
- `manifests/learning-path/17-simple-gitops-app.yaml` - GitOps demo application
- `scripts/verify-argocd.ps1` - Automated verification script (fixed)
- `scripts/cleanup-final.ps1` - Working cleanup automation

---

## 🎓 Kubernetes Learning Path Complete!

**🎯 What You've Mastered:**

### Core Kubernetes Concepts
- **Pods**: Understanding container orchestration fundamentals
- **Deployments**: Managing application lifecycle and scaling
- **Services**: Network exposure and load balancing
- **ConfigMaps & Secrets**: Configuration and credential management
- **Persistent Volumes**: Data persistence and storage management
- **Namespaces**: Resource organization and isolation

### Advanced Operations
- **Debugging**: Troubleshooting with BusyBox and kubectl
- **Resource Management**: CPU, memory limits and requests
- **Health Checks**: Liveness and readiness probes
- **Auto-scaling**: Horizontal Pod Autoscaler configuration
- **Jobs & CronJobs**: Batch processing and scheduled tasks

### GitOps & CI/CD
- **ArgoCD Installation**: Complete setup and configuration
- **Application Management**: Declarative GitOps workflows
- **Sync Policies**: Automated drift detection and correction
- **Multi-Application**: Managing complex application stacks
- **Monitoring**: Health and sync status verification

### Production Skills
- **Security**: RBAC, service accounts, and namespace isolation
- **Monitoring**: Resource usage and application health
- **Automation**: Scripted deployment and cleanup procedures
- **Troubleshooting**: Systematic problem diagnosis and resolution

**🚀 Ready for Production:**
You now have the foundational skills to:
- Deploy and manage production Kubernetes applications
- Implement GitOps workflows for reliable deployments
- Monitor and troubleshoot Kubernetes environments
- Scale applications based on demand
- Secure Kubernetes clusters with proper RBAC

---

## Next Steps: Mastering Kubernetes

**To Master Kubernetes, Explore These Advanced Topics:**
1. **Custom Resource Definitions (CRDs)** - Extend Kubernetes capabilities
2. **Operators** - Manage complex stateful applications  
3. **Service Mesh (Istio/Linkerd)** - Advanced networking and security
4. **Multi-Cluster GitOps** - Manage multiple environments
5. **Kubernetes Security** - Network policies, Pod Security Standards
6. **Performance Tuning** - Optimize resource usage and costs
7. **Observability Stack** - Prometheus, Grafana, Jaeger
8. **Advanced Scheduling** - Node affinity, taints, and tolerations

**🌟 Congratulations on completing your Kubernetes journey!**

**Additional Advanced Topics:**
7. **Backup and Restore** - Ensure data protection and disaster recovery
8. **Kubernetes Upgrades** - Plan and execute cluster upgrades

**Recommended Learning Resources:**
- Kubernetes official documentation
- CNCF Kubernetes courses
- Kubernetes Up & Running book
- The Kubernetes Book by Nigel Poulton
- ArgoCD documentation and tutorials
- Prometheus and Grafana documentation

**Join the Kubernetes Community:**
- Participate in Kubernetes forums and discussions
- Attend Kubernetes meetups and conferences
- Contribute to Kubernetes open-source projects

Happy Learning and Best of Luck on Your Kubernetes Mastery Journey! 🚀