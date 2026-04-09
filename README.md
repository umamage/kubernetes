# Clothing Classifier - Kubernetes Deployment Guide

This guide contains all the commands needed to set up and deploy the clothing classifier application using Kubernetes, Docker, and related tools.

## Table of Contents
1. [kubectl Installation](#kubectl-installation)
2. [kind Installation](#kind-installation)
3. [Running the Application Locally](#running-the-application-locally)
4. [Docker Image Build](#docker-image-build)
5. [Kubernetes Cluster Setup](#kubernetes-cluster-setup)
6. [Deployment](#deployment)
7. [Accessing the Application](#accessing-the-application)

---

## kubectl Installation

### Step 1: Check current installation
```bash
which kubectl
ls -la /usr/local/bin/kubectl
kubectl version --client
```

### Step 2: Download kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

### Step 3: Make it executable and install
```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Step 4: Verify installation
```bash
kubectl version --client
```

---

## kind Installation

### Step 1: Download kind
**Note:** Always download to a writable directory first, not directly to system directories.
```bash
curl -Lo kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
```

### Step 2: Make it executable and install
```bash
chmod +x kind
sudo mv kind /usr/local/bin/
```

### Step 3: Verify installation
```bash
kind version
```

---

## Running the Application Locally

### Prerequisites
- Python 3.12+
- uv package manager
- Dependencies listed in `pyproject.toml`

### Start the development server
```bash
cd /workspaces/kubernetes
uv run uvicorn app:app --host 0.0.0.0 --port 8080 --reload
```

### Test the application
```bash
curl -s http://localhost:8080/ | jq .
```

---

## Docker Image Build

### Build the Docker image
```bash
cd /workspaces/kubernetes
docker build -t clothing-classifier:v1 .
```

**Dockerfile Contents:**
- Base image: `python:3.13.5-slim-bookworm`
- Includes: uv, Python dependencies, model files
- Exposes port 8080
- Entry point: uvicorn server

---

## Kubernetes Cluster Setup

### Create a kind cluster (local Kubernetes)
```bash
kind create cluster --name clothing-cluster
```

### Alternative: Create another cluster
```bash
kind create cluster --name mlzoomcamp
```

### Check existing clusters
```bash
kind get clusters
```

### Set kubectl context
```bash
kubectl cluster-info --context kind-clothing-cluster
```

---

## Deployment

### Step 1: Load Docker image into the cluster
```bash
kind load docker-image clothing-classifier:v1 --name clothing-cluster
```

### Step 2: Apply the Kubernetes deployment
```bash
cd /workspaces/kubernetes/k8s
kubectl apply -f deployment.yaml
```

### Step 3: Check deployment status
```bash
kubectl get pods
kubectl get pods -l app=clothing-classifier
```

### Step 4: View application logs
```bash
kubectl logs deployment/clothing-classifier --follow
```

### Step 5: View logs from specific pod
```bash
kubectl logs <pod-name>
```

---

## Accessing the Application

### Step 1: Set up port forwarding
```bash
kubectl port-forward deployment/clothing-classifier 8080:8080
```

### Step 2: Test the endpoints

#### Root endpoint
```bash
curl -s http://localhost:8080/ | jq .
```

Expected response:
```json
{
  "message": "Clothing Classification Service"
}
```

#### Health check endpoint
```bash
curl -s http://localhost:8080/health | jq .
```

#### Prediction endpoint
```bash
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/clothing-image.jpg"}'
```

---

## Deployment Configuration

The deployment is configured in `k8s/deployment.yaml` with:
- **Replicas**: 2 for high availability
- **Image**: `clothing-classifier:v1`
- **Pull Policy**: Never (uses locally loaded image)
- **Port**: 8080
- **Resource Requests**: 250m CPU, 256Mi memory
- **Resource Limits**: 500m CPU, 512Mi memory
- **Liveness Probe**: HTTP GET on `/health` (initialDelaySeconds: 10, periodSeconds: 10)
- **Readiness Probe**: HTTP GET on `/health` (initialDelaySeconds: 5, periodSeconds: 5)

---

## Useful kubectl Commands

```bash
# Get all resources
kubectl get all

# Get deployments
kubectl get deployments

# Get services
kubectl get services

# Get pod details
kubectl describe pod <pod-name>

# Delete deployment
kubectl delete deployment clothing-classifier

# Scale deployment
kubectl scale deployment clothing-classifier --replicas=3

# Delete cluster
kind delete cluster --name clothing-cluster
```

---

## Troubleshooting

### Permission denied when installing tools
- Always download to a user-writable directory first (e.g., current directory, `/tmp`, home directory)
- Then use `sudo` to move to `/usr/local/bin`

### Port already in use
```bash
lsof -i :8080
kill <PID>
```

### Cluster context issue
```bash
kubectl config get-contexts
kubectl config use-context kind-clothing-cluster
```

### Pod not ready
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

---

## Notes

- The clothing model (`clothing-model.onnx`) is pre-included in the repository
- The application uses ONNX Runtime for inference
- Image preprocessing follows PyTorch conventions with standard ImageNet normalization
- All commands assume you're in the `/workspaces/kubernetes` directory unless otherwise specified
