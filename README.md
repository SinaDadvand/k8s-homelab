<!-- filepath: c:\Users\sinad\VS Code\homelab-prj\README.md -->
# K8s Home Lab Setup with Kind

This repository contains the configuration and scripts to set up a Kubernetes home lab using Kind (Kubernetes in Docker).

## Prerequisites

Before you begin, ensure you have the following installed:

*   [Docker](https://docs.docker.com/get-docker/)
*   [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
*   [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

## Initial Setup

1.  **Clone the repository (if you haven't already):**
    ```bash
    git clone <repository-url>
    cd homelab-prj
    ```

2.  **Navigate to the `homelab` directory:**
    ```bash
    cd homelab
    ```

3.  **Create the Kind cluster using the declarative configuration:**
    ```bash
    kind create cluster --config .\kind-cluster.yaml
    ```
    
    **Note**: The `kind-cluster.yaml` file includes `extraPortMappings` configuration to expose NodePort services on localhost (see [NodePort Access on Windows](#nodeport-access-on-windows) section below).

4.  **Verify the cluster:**
    After the cluster is created, you can verify that your cluster is running and `kubectl` is configured correctly:
    ```bash
    kubectl cluster-info --context kind-homelab
    kubectl get nodes -o wide
    ```
    You should see your control-plane and two worker nodes listed.

5.  **Deploy a sample application (Optional):**
    To test your cluster, you can deploy the provided sample application.
    Navigate to the `manifests/sample-app` directory:
    ```bash
    cd manifests/sample-app
    ```
    Apply the manifests:
    ```bash
    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml
    ```
    Check the status of the deployment and service:
    ```bash
    kubectl get deployments
    kubectl get services
    kubectl get pods
    ```
    
    **Access the application:**
    - **Via NodePort (Windows)**: Thanks to the `extraPortMappings` in our Kind configuration, you can access the service directly at `http://localhost:30001`
    - **Via Port Forwarding (Alternative)**: You can also forward a local port to the service:
      ```bash
      kubectl port-forward service/hello-world-service 8080:80
      ```
      Then open your browser and go to `http://localhost:8080`.

## NodePort Access on Windows

### The Issue
By default, Kind on Windows doesn't automatically expose NodePort services on localhost. This happens because:
- Kind runs Kubernetes nodes as Docker containers
- Docker networking on Windows doesn't automatically bridge NodePort services to the host machine
- Services work fine inside the cluster but aren't accessible from the Windows host

### The Solution
Our `kind-cluster.yaml` configuration includes `extraPortMappings` to solve this issue:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: homelab
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30001
    hostPort: 30001
    protocol: TCP
- role: worker
- role: worker
```

This configuration:
- Maps the container port 30001 (NodePort) to host port 30001
- Enables direct access to NodePort services via `localhost:30001`
- Works specifically for the sample application's NodePort service

### Alternative Solutions
If you need to expose additional NodePort services, you can:
1. **Add more port mappings** to the `extraPortMappings` section
2. **Use port forwarding** with `kubectl port-forward` for individual services
3. **Use kubectl proxy** to access services through the Kubernetes API

## Managing Kind Clusters

### List available clusters:
```bash
kind get clusters
```

### Switch between cluster contexts:
```bash
kubectl config use-context kind-{cluster_name}
```

### Create alias for kubectl (PowerShell):
```powershell
Set-Alias -Name k -Value kubectl
```

## Next Steps

*   Explore deploying other applications to your cluster.
*   Learn more about managing your Kind cluster using the [Kind documentation](https://kind.sigs.k8s.io/docs/user/quick-start/).
*   Customize the `homelab/kind-cluster.yaml` file to change cluster configuration (e.g., add more nodes, configure additional port mappings).
*   Set up additional tools like Helm, monitoring, or service mesh for a more complete lab environment.

## Troubleshooting

### Common Issues:

1. **Service not accessible on localhost:30001**
   - Ensure the Kind cluster was created with the `kind-cluster.yaml` configuration
   - Verify the service is using NodePort type with nodePort: 30001
   - Check if the pod is running: `kubectl get pods`

2. **Docker port binding errors**
   - Check if port 5001 (registry) or 30001 is already in use
   - Try changing the port numbers in the configuration
   - Restart Docker Desktop if needed

3. **kubectl context issues**
   - Verify you're using the correct context: `kubectl config current-context`
   - Switch to homelab context: `kubectl config use-context kind-homelab`

## Cleaning Up

To delete the Kind cluster:
```bash
kind delete cluster --name homelab
```

To delete the local Docker registry:
```bash
docker rm -f kind-registry
```
