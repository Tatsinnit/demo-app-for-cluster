# AKS Hello World Deployment Plan

## Objective
Deploy a minimal Hello World web app to an AKS cluster using:
- A tiny Python Flask app
- Docker image in Azure Container Registry (ACR)
- Kubernetes Deployment and Service manifests

This plan is executable in sequence.

## Folder Structure
Create and keep this structure:

```text
demo-app-for-cluster/
  app/
    app.py
    requirements.txt
  k8s/
    namespace.yaml
    deployment.yaml
    service.yaml
  Dockerfile
  .dockerignore
  plan.md
```

## Prerequisites
- Azure CLI installed and logged in
- Docker installed and running
- kubectl installed
- An Azure subscription with permissions to create resources

Optional but recommended:
- Use one resource token for consistent naming (5 lowercase alphanumeric chars)
- Keep Kubernetes names lowercase and under 20 chars

## Step 0: Set Variables
Run this first in terminal and replace values where needed.

```bash
# Required values
export LOCATION="eastus"
export RESOURCE_GROUP="rgakshello01"
export AKS_NAME="akshello01"
export ACR_NAME="acrhello01"
export NAMESPACE="hello"
export APP_NAME="hello-app"
export IMAGE_TAG="v1"

# Auto
export IMAGE_NAME="$ACR_NAME.azurecr.io/$APP_NAME:$IMAGE_TAG"
```

Validation:
- ACR name must be globally unique and alphanumeric only
- Keep all K8s names lowercase with hyphen or digits

## Step 1: Create Minimal App
Create app files.

### app/app.py
```python
from flask import Flask

app = Flask(__name__)

@app.get("/")
def hello():
    return {"message": "hello from aks"}

@app.get("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

### app/requirements.txt
```txt
flask==3.0.3
```

## Step 2: Create Docker Assets
### Dockerfile
```Dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

### .dockerignore
```dockerignore
.git
.gitignore
__pycache__
*.pyc
*.pyo
*.pyd
.env
.venv
```

## Step 3: Create Azure Resources
```bash
az account show >/dev/null
az group create --name "$RESOURCE_GROUP" --location "$LOCATION"
az acr create --resource-group "$RESOURCE_GROUP" --name "$ACR_NAME" --sku Basic
az aks create --resource-group "$RESOURCE_GROUP" --name "$AKS_NAME" --node-count 1 --generate-ssh-keys
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_NAME" --overwrite-existing
az aks update --resource-group "$RESOURCE_GROUP" --name "$AKS_NAME" --attach-acr "$ACR_NAME"
```

Checkpoint:
```bash
kubectl config current-context
kubectl get nodes
```

## Step 4: Build and Push Image
```bash
az acr login --name "$ACR_NAME"
docker build -t "$IMAGE_NAME" .
docker push "$IMAGE_NAME"
```

Checkpoint:
```bash
az acr repository show-tags --name "$ACR_NAME" --repository "$APP_NAME" --output table
```

## Step 5: Create Kubernetes Manifests
### k8s/namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hello
```

### k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: hello-app
        image: REPLACE_IMAGE
        ports:
        - containerPort: 8000
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
```

### k8s/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  namespace: hello
spec:
  selector:
    app: hello-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

Replace image placeholder and apply manifests:

```bash
kubectl apply -f k8s/namespace.yaml
sed "s|REPLACE_IMAGE|$IMAGE_NAME|g" k8s/deployment.yaml | kubectl apply -f -
kubectl apply -f k8s/service.yaml
```

## Step 6: Verify Deployment
```bash
kubectl get pods -n "$NAMESPACE"
kubectl rollout status deployment/hello-app -n "$NAMESPACE"
kubectl get svc hello-svc -n "$NAMESPACE" -w
```

When EXTERNAL-IP is assigned:

```bash
export APP_URL="http://$(kubectl get svc hello-svc -n "$NAMESPACE" -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
curl "$APP_URL/"
curl "$APP_URL/health"
```

Expected responses:
- / returns hello message JSON
- /health returns status ok JSON

## Step 7: Cleanup (Optional)
```bash
az group delete --name "$RESOURCE_GROUP" --yes --no-wait
```

## Optional Next Step: Argo CD (GitOps)
After manual deployment works:
- Commit app + k8s manifests to repo
- Install Argo CD in cluster
- Create an Argo CD Application pointing to k8s path
- Sync and validate

## Common Issues
1. Image pull error:
- Ensure AKS is attached to ACR
- Ensure image path is correct in deployment

2. No external IP:
- Wait a few minutes for load balancer provisioning
- Check service events: `kubectl describe svc hello-svc -n hello`

3. Pod crash loop:
- Check logs: `kubectl logs deploy/hello-app -n hello`
- Check probes and app port (8000)
