# Kubernetes Deployment Guide

This guide provides step-by-step instructions for containerizing and deploying the Node.js application using different Kubernetes deployment strategies with Argo CD.

## Prerequisites

- Docker installed and configured
- Kubernetes cluster (minikube, kind, or cloud provider)
- kubectl configured to access your cluster
- Docker Hub account
- Git repository access

## 1. Docker Setup

### Build and Push Docker Image

**Important:** Replace `your-dockerhub-username` with your actual Docker Hub username in all YAML files before proceeding.

```bash
# Navigate to the project root directory
cd ..

# Build the Docker image using the Dockerfile in deployment/
docker build -f deployment/Dockerfile -t your-dockerhub-username/node-express-app:v1.0.0 .

# Login to Docker Hub
docker login

# Push the image
docker push your-dockerhub-username/node-express-app:v1.0.0

# Tag and push as latest
docker tag your-dockerhub-username/node-express-app:v1.0.0 your-dockerhub-username/node-express-app:latest
docker push your-dockerhub-username/node-express-app:latest

# Build v1.1.0 for testing deployments
docker tag your-dockerhub-username/node-express-app:v1.0.0 your-dockerhub-username/node-express-app:v1.1.0
docker push your-dockerhub-username/node-express-app:v1.1.0
```

## 2. Kubernetes Cluster Setup

### Start Local Cluster (Choose one)

**Option A: Minikube**
```bash
minikube start
minikube addons enable ingress
```

**Option B: Kind**
```bash
kind create cluster --name deployment-demo
```

## 3. Argo CD Installation

```bash
# Create Argo CD namespace
kubectl create namespace argocd

# Install Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for Argo CD to be ready (this may take a few minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get initial admin password
echo "Argo CD Admin Password:"
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Access Argo CD UI (run in separate terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access at: https://localhost:8080
# Username: admin
# Password: (from command above)
```

## 4. Deploy Applications with Argo CD

### Apply Argo CD Applications

```bash
# Apply all Argo CD applications
kubectl apply -f argocd/

# Verify applications are created
kubectl get applications -n argocd

# Check application status
kubectl describe application node-app-blue-green -n argocd
kubectl describe application node-app-rolling-update -n argocd
kubectl describe application node-app-canary -n argocd
```

### Manual Sync (if auto-sync is disabled)

```bash
# Install Argo CD CLI (optional)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login to Argo CD CLI
argocd login localhost:8080

# Sync applications
argocd app sync node-app-blue-green
argocd app sync node-app-rolling-update
argocd app sync node-app-canary
```

## 5. Testing Deployments

### Blue-Green Deployment Testing

```bash
# Check blue deployment
kubectl get pods -l app=node-app,version=blue
kubectl get pods -l app=node-app,version=green

# Test blue environment
kubectl port-forward svc/node-app-service 3000:80
curl http://localhost:3000/api

# Switch to green environment
kubectl patch service node-app-service -p '{"spec":{"selector":{"version":"green"}}}'

# Scale up green deployment first
kubectl scale deployment node-app-green --replicas=3

# Wait for green pods to be ready
kubectl wait --for=condition=available deployment/node-app-green --timeout=300s

# Test green environment
curl http://localhost:3000/api

# Rollback to blue if needed
kubectl patch service node-app-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Rolling Update Testing

```bash
# Check current deployment
kubectl get pods -l app=node-app,strategy=rolling

# Test rolling update service
kubectl port-forward svc/node-app-rolling-service 3001:80
curl http://localhost:3001/api

# Update deployment image
kubectl set image deployment/node-app-rolling node-app=your-dockerhub-username/node-express-app:v1.1.0

# Watch rolling update in action
kubectl rollout status deployment/node-app-rolling

# Rollback if needed
kubectl rollout undo deployment/node-app-rolling
```

### Canary Deployment Testing

```bash
# Check stable and canary deployments
kubectl get pods -l app=node-app,version=stable
kubectl get pods -l app=node-app,version=canary

# Test canary service (distributes traffic between stable and canary)
kubectl port-forward svc/node-app-canary-service 3002:80

# Make multiple requests to see traffic distribution
for i in {1..10}; do curl http://localhost:3002/api; done

# Scale canary deployment for more traffic
kubectl scale deployment node-app-canary --replicas=2

# Promote canary to stable (replace stable deployment)
kubectl set image deployment/node-app-stable node-app=your-dockerhub-username/node-express-app:v1.1.0
kubectl scale deployment node-app-canary --replicas=0
```

## 6. Monitoring and Verification

```bash
# Check all deployments
kubectl get deployments -A

# Check all services
kubectl get services -A

# Check pod logs
kubectl logs -l app=node-app --tail=50

# Check Argo CD application health
kubectl get applications -n argocd

# Watch pods in real-time
kubectl get pods -w -l app=node-app
```

## 7. Cleanup

```bash
# Delete Argo CD applications
kubectl delete -f argocd/

# Delete remaining resources
kubectl delete deployment,service -l app=node-app

# Delete Argo CD
kubectl delete namespace argocd

# Stop cluster (if using local cluster)
minikube stop  # or kind delete cluster --name deployment-demo
```

## Important Notes

1. **Update Image Names**: Replace `your-dockerhub-username` with your actual Docker Hub username in all YAML files.

2. **Repository URL**: Update the repository URL in Argo CD applications to match your Git repository.

3. **Namespace**: All applications deploy to the `default` namespace. Modify if different namespace is required.

4. **Resource Limits**: Adjust CPU and memory limits based on your cluster capacity.

5. **Health Checks**: The applications include liveness and readiness probes for better reliability.

## Deployment Strategies Summary

- **Blue-Green**: Zero downtime with instant rollback capability
- **Rolling Update**: Gradual deployment with resource efficiency
- **Canary**: Risk mitigation by testing with subset of traffic

Each strategy serves different use cases and risk tolerance levels.
