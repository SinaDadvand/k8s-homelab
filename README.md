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

3.  **Make the setup script executable:**
    ```bash
    chmod +x scripts/setup-kind.sh
    ```

4.  **Run the setup script:**
    This script will:
    *   Create a local Docker registry (if it doesn't already exist).
    *   Create a Kind cluster named `homelab` with one control-plane node and two worker nodes.
    *   Configure the Kind cluster to use the local registry.
    *   Document the local registry in the cluster.

    ```bash
    ./scripts/setup-kind.sh
    ```

5.  **Verify the cluster:**
    After the script completes, you can verify that your cluster is running and `kubectl` is configured correctly:
    ```bash
    kubectl cluster-info --context kind-homelab
    kubectl get nodes
    ```
    You should see your control-plane and worker nodes listed.

6.  **Deploy a sample application (Optional):**
    To test your cluster and registry, you can deploy the provided sample application.
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
    Once the `hello-world-service` has an external IP (it might take a moment with Kind's LoadBalancer), you can try accessing it. For Kind, you might need to forward a local port to the service:
    ```bash
    kubectl port-forward service/hello-world-service 8080:80
    ```
    Then open your browser and go to `http://localhost:8080`.

## Next Steps

*   Explore deploying other applications to your cluster.
*   Learn more about managing your Kind cluster using the [Kind documentation](https://kind.sigs.k8s.io/docs/user/quick-start/).
*   Customize the `homelab/kind-cluster.yaml` file to change cluster configuration (e.g., add more nodes, configure port mappings).

## Cleaning Up

To delete the Kind cluster:
```bash
kind delete cluster --name homelab
```

To delete the local Docker registry:
```bash
docker rm -f kind-registry
```
