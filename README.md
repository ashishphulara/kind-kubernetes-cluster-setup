# Kubernetes in Docker (Kind) Setup & Operations Guide

A complete, practical guide to installing, configuring, and managing local Kubernetes clusters using **Kind (Kubernetes in Docker)**. This repository includes ready-to-use cluster configurations, deployment manifests, and workflows for local development and testing.

---

## Table of Contents

- [Overview](#overview)
- [Why Kind?](#why-kind)
- [Prerequisites](#prerequisites)
- [Installation Guide](#installation-guide)
  - [1. Install Docker](#1-install-docker)
  - [2. Install kubectl](#2-install-kubectl)
  - [3. Install Kind](#3-install-kind)
- [Quick Start: Single-Node Cluster](#quick-start-single-node-cluster)
- [Managing Multiple Clusters](#managing-multiple-clusters)
- [Advanced Cluster Configurations](#advanced-cluster-configurations)
  - [Multi-Node Cluster (1 Master + 2 Workers)](#multi-node-cluster-1-master--2-workers)
  - [Targeting Specific Kubernetes Versions](#targeting-specific-kubernetes-versions)
  - [Port Mapping (Host to Container)](#port-mapping-host-to-container)
  - [Mounting Host Directories into Nodes](#mounting-host-directories-into-nodes)
- [Working with Local Docker Images](#working-with-local-docker-images)
- [Configuring Ingress (NGINX Ingress Controller)](#configuring-ingress-nginx-ingress-controller)
- [Hands-on Walkthrough: Deploying a Sample App](#hands-on-walkthrough-deploying-a-sample-app)
- [Useful Commands Cheatsheet](#useful-commands-cheatsheet)
- [Troubleshooting & FAQs](#troubleshooting--faqs)
- [Repository Structure](#repository-structure)

---

## Overview

**Kind (Kubernetes in Docker)** is an open-source tool developed under the Kubernetes SIGs umbrella. It runs local Kubernetes clusters by running each Kubernetes node (control-plane and workers) as a separate Docker container.

```
       Host Machine (Your PC / Laptop)
+-----------------------------------------------+
| Docker Engine                                 |
|  +-----------------------------------------+  |
|  | Kind Node (Container: Control Plane)   |  |
|  | - API Server, etcd, CoreDNS, Scheduler  |  |
|  +-----------------------------------------+  |
|  +-----------------------------------------+  |
|  | Kind Node (Container: Worker 1)         |  |
|  | - Kubelet, Containerd, Pods             |  |
|  +-----------------------------------------+  |
|  +-----------------------------------------+  |
|  | Kind Node (Container: Worker 2)         |  |
|  | - Kubelet, Containerd, Pods             |  |
|  +-----------------------------------------+  |
+-----------------------------------------------+
```

---

## Why Kind?

| Feature | Kind | Minikube | K3s / K3d |
| :--- | :--- | :--- | :--- |
| **Node Architecture** | Docker containers | VM or Container | Container (lightweight K3s binary) |
| **Multi-Node Clusters** | Seamless via simple YAML | Complex / Resource heavy | Supported |
| **Startup Speed** | ~30 - 60 seconds | 1 - 3 minutes | ~30 seconds |
| **CI/CD Integration** | Excellent (GitHub Actions / GitLab CI) | Average | Excellent |
| **Local Images** | `kind load docker-image` | `minikube image load` | `k3d image import` |
| **Upstream Conformity** | Certified Kubernetes conformant | Certified | Lightweight alternative distribution |

---

## Prerequisites

Before using Kind, ensure you have:

1. **Docker**: Running Docker Engine or Docker Desktop with container runtime enabled.
   - Recommended resources allocated in Docker Desktop:
     - **CPU**: 2+ cores (4+ cores recommended for multi-node).
     - **RAM**: 4 GB minimum (8 GB recommended for multi-node + Ingress).
2. **Terminal / Shell**: PowerShell or WSL2 (Windows), Bash / Zsh (macOS / Linux).
3. **Administrative / Sudo privileges** for package installation.

---

## Installation Guide

### 1. Install Docker

- **Windows / macOS**: Download and install [Docker Desktop](https://www.docker.com/products/docker-desktop/).
  - *Windows tip*: Enable the WSL 2 backend in Docker Desktop settings for optimal performance.
- **Linux (Ubuntu/Debian)**:
  ```bash
  sudo apt-get update
  sudo apt-get install -y docker.io
  sudo usermod -aG docker $USER
  newgrp docker
  ```
Verify Docker is running:
```bash
docker version
```

---

### 2. Install kubectl

`kubectl` is the Kubernetes command-line tool used to interact with clusters.

#### Windows
- **Via winget**:
  ```powershell
  winget install Kubernetes.kubectl
  ```
- **Via Chocolatey**:
  ```powershell
  choco install kubernetes-cli
  ```
- **Via Scoop**:
  ```powershell
  scoop install kubectl
  ```

#### macOS
- **Via Homebrew**:
  ```bash
  brew install kubectl
  ```

#### Linux
- **Via curl**:
  ```bash
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin/
  ```

Verify `kubectl` installation:
```bash
kubectl version --client
```

---

### 3. Install Kind

#### Windows
- **Via winget**:
  ```powershell
  winget install Kubernetes.kind
  ```
- **Via Chocolatey**:
  ```powershell
  choco install kind
  ```
- **Via Scoop**:
  ```powershell
  scoop install kind
  ```
- **Manual Binary (PowerShell)**:
  ```powershell
  curl.exe -Lo kind.exe https://kind.sigs.k8s.io/dl/v0.27.0/kind-windows-amd64
  # Move to a directory in your PATH (e.g. C:\Windows\System32 or a custom bin directory)
  Move-Item .\kind.exe C:\Windows\System32\kind.exe
  ```

#### macOS
- **Via Homebrew**:
  ```bash
  brew install kind
  ```

#### Linux
- **Via curl**:
  ```bash
  # For AMD64 / x86_64
  [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
  # For ARM64
  [ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-arm64
  chmod +x ./kind
  sudo mv ./kind /usr/local/bin/kind
  ```

Verify Kind installation:
```bash
kind version
```

---

## Quick Start: Single-Node Cluster

### Step 1: Create a default cluster
```bash
kind create cluster
```
By default, this:
- Downloads the latest node image.
- Creates a single-node cluster named `kind`.
- Configures your `kubectl` context to `kind-kind`.

### Step 2: Check cluster status
```bash
# Check cluster endpoints
kubectl cluster-info

# View nodes
kubectl get nodes -o wide

# View system pods
kubectl get pods -A
```

### Step 3: Delete the cluster
```bash
kind delete cluster
```

---

## Managing Multiple Clusters

You can run multiple isolated Kind clusters concurrently by assigning distinct names:

```bash
# Create two distinct clusters
kind create cluster --name dev-cluster
kind create cluster --name staging-cluster

# List active Kind clusters
kind get clusters

# List kubectl contexts
kubectl config get-contexts

# Switch context between clusters
kubectl config use-context kind-dev-cluster
kubectl config use-context kind-staging-cluster

# Delete a specific cluster
kind delete cluster --name dev-cluster
kind delete cluster --name staging-cluster
```

---

## Advanced Cluster Configurations

Kind clusters are configured declaratively using YAML files (`apiVersion: kind.x-k8s.io/v1alpha4`).

### Multi-Node Cluster (1 Master + 2 Workers)

Use [configs/multi-node-cluster.yaml](file:///c:/Users/dell/Desktop/kind-kubernetes-cluster-setup/configs/multi-node-cluster.yaml):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

**Create the multi-node cluster:**
```bash
kind create cluster --name multi-node --config configs/multi-node-cluster.yaml
```

**Verify the nodes:**
```bash
kubectl get nodes
```
*Output:*
```text
NAME                         STATUS   ROLES           AGE   VERSION
multi-node-control-plane     Ready    control-plane   45s   v1.31.0
multi-node-worker            Ready    <none>          32s   v1.31.0
multi-node-worker2           Ready    <none>          32s   v1.31.0
```

---

### Targeting Specific Kubernetes Versions

Kind allows testing against older or specific Kubernetes versions by specifying the pre-built node image:

```bash
# Example: Kubernetes v1.29.2
kind create cluster --image kindest/node:v1.29.2

# Example: Kubernetes v1.30.2
kind create cluster --image kindest/node:v1.30.2
```

> Node image tags are published on [Docker Hub (`kindest/node`)](https://hub.docker.com/r/kindest/node/tags).

---

### Port Mapping (Host to Container)

To test services or access NodePorts directly from your host's browser (`localhost:8080`), map host ports to the container nodes:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP
```

Now, any Kubernetes `NodePort` service exposed on `30080` is reachable on `http://localhost:8080`.

---

### Mounting Host Directories into Nodes

To share files from your host into the Kubernetes nodes (e.g. for persistent storage testing):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /path/on/host
        containerPath: /var/data
```

---

## Working with Local Docker Images

When developing applications locally, you build Docker images that are not pushed to Docker Hub or an external registry. If you reference them in your pods, you will encounter `ImagePullBackOff` or `ErrImagePull`.

Kind provides the `kind load` command to load local Docker images directly into the cluster's nodes.

### 1. Build your local Docker image
```bash
docker build -t my-app:v1.0 .
```

### 2. Load the image into the Kind cluster
```bash
kind load docker-image my-app:v1.0 --name <cluster-name>
```

### 3. Configure your Pod / Deployment YAML
Ensure `imagePullPolicy` is set to `IfNotPresent` or `Never`:
```yaml
spec:
  containers:
    - name: my-app
      image: my-app:v1.0
      imagePullPolicy: IfNotPresent
```

---

## Configuring Ingress (NGINX Ingress Controller)

To route HTTP/HTTPS traffic from `localhost` into your cluster using Ingress, the Kind cluster needs two things:
1. Host port mappings for ports `80` and `443`.
2. The `ingress-ready=true` node label on the control-plane node.

This is pre-configured in [configs/ingress-cluster.yaml](file:///c:/Users/dell/Desktop/kind-kubernetes-cluster-setup/configs/ingress-cluster.yaml):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
```

### Deploy Ingress Controller:

1. **Create the cluster:**
   ```bash
   kind create cluster --name ingress-cluster --config configs/ingress-cluster.yaml
   ```

2. **Apply the Kind-specific NGINX Ingress Controller manifest:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
   ```

3. **Wait for the Ingress pods to be ready:**
   ```bash
   kubectl wait --namespace ingress-nginx \
     --for=condition=ready pod \
     --selector=app.kubernetes.io/component=controller \
     --timeout=120s
   ```

---

## Hands-on Walkthrough: Deploying a Sample App

Let's put everything together by deploying an NGINX sample application and routing traffic to it via Ingress.

### Step 1: Ensure Ingress cluster is running
```bash
kind create cluster --name ingress-demo --config configs/ingress-cluster.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
```

### Step 2: Deploy the sample application
Apply [examples/sample-app.yaml](file:///c:/Users/dell/Desktop/kind-kubernetes-cluster-setup/examples/sample-app.yaml):
```bash
kubectl apply -f examples/sample-app.yaml
```

This creates:
- A `Deployment` with 2 replicas of NGINX.
- A `Service` of type `ClusterIP` exposing port 80.
- An `Ingress` rule routing path `/` to `demo-service`.

### Step 3: Check resources
```bash
kubectl get deployments,pods,services,ingress
```

### Step 4: Test in your browser or curl
```bash
curl http://localhost/
```
You should see the default **"Welcome to nginx!"** HTML response!

### Step 5: Clean up
```bash
kind delete cluster --name ingress-demo
```

---

## Useful Commands Cheatsheet

### Kind CLI
| Command | Description |
| :--- | :--- |
| `kind create cluster` | Creates default single-node cluster named `kind` |
| `kind create cluster --name <name>` | Creates a cluster with a custom name |
| `kind create cluster --config <file.yaml>` | Creates a cluster using a configuration file |
| `kind get clusters` | Lists all active Kind clusters |
| `kind get nodes` | Lists all node containers for active clusters |
| `kind load docker-image <img:tag>` | Loads a local Docker image into the cluster nodes |
| `kind export logs ./logs` | Exports cluster logs for debugging and troubleshooting |
| `kind delete cluster` | Deletes the default cluster |
| `kind delete cluster --name <name>` | Deletes a named cluster |
| `kind delete clusters --all` | Deletes all Kind clusters |

### Kubectl Essentials
| Command | Description |
| :--- | :--- |
| `kubectl cluster-info` | Displays cluster control plane and CoreDNS addresses |
| `kubectl get nodes -o wide` | Displays all nodes with IPs, OS image, and container runtime |
| `kubectl get pods -A` | Lists all pods across all namespaces |
| `kubectl get svc` | Lists services in the current namespace |
| `kubectl logs <pod-name>` | Streams logs from a pod container |
| `kubectl describe pod <pod-name>` | Detailed info and recent events for debugging pods |
| `kubectl exec -it <pod-name> -- sh` | Opens an interactive shell inside a container |
| `kubectl port-forward svc/<name> 8080:80` | Forwards a local port to a cluster service directly |

---

## Troubleshooting & FAQs

### 1. `docker: command not found` or `Cannot connect to the Docker daemon`
- **Cause**: Docker is either not installed or not running.
- **Fix**: Launch Docker Desktop (Windows/Mac) or run `sudo systemctl start docker` (Linux). Verify with `docker ps`.

### 2. `failed to create cluster: ... port is already allocated`
- **Cause**: Another service (like IIS, Apache, or a previous Kind cluster) is already listening on port 80 or 443.
- **Fix**:
  - Stop any existing containers using the port: `docker ps` followed by `docker stop <container_id>`.
  - Alternatively, change the `hostPort` in your YAML config to an unused port (e.g. `8080`).

### 3. Pods stuck in `ImagePullBackOff` for local images
- **Cause**: Kubernetes is attempting to pull your local image from a remote registry.
- **Fix**:
  1. Load the image into Kind: `kind load docker-image <image-name>:<tag> --name <cluster-name>`.
  2. Set `imagePullPolicy: IfNotPresent` or `Never` in your pod spec.

### 4. WSL 2 Docker Desktop integration issues (Windows)
- In Docker Desktop, navigate to **Settings** > **Resources** > **WSL Integration**.
- Ensure **Enable integration with my default WSL distro** is checked, and toggle on your specific distro (e.g., Ubuntu).

### 5. Cleaning up orphan Kind containers and networks
If a cluster deletion gets interrupted:
```bash
# List and remove all kind Docker containers
docker ps -a --filter "name=kind-" --format "{{.ID}}" | ForEach-Object { docker rm -f $_ }

# Clean up dangling kind network
docker network rm kind
```

---

## Repository Structure

```text
kind-kubernetes-cluster-setup/
├── configs/
│   ├── multi-node-cluster.yaml    # 1 Control plane + 2 Worker nodes configuration
│   └── ingress-cluster.yaml       # Cluster with port 80/443 mapping and ingress label
├── examples/
│   └── sample-app.yaml            # NGINX Deployment, Service, and Ingress manifest
└── README.md                      # Comprehensive guide and documentation
```

---

## License

This project is open-source and available under the [MIT License](LICENSE).
