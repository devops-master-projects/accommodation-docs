# Complete Step-by-Step Guide: Mac Setup & Frontend Deployment

## Part 1: Mac Installation (Prerequisites)

### Step 1.1: Install Homebrew (if not installed)
```bash
# Check if Homebrew is installed
which brew

# If not installed, install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Follow the instructions to add Homebrew to your PATH
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### Step 1.2: Install Docker Desktop
```bash
# Install Docker Desktop
brew install --cask docker

# After installation, open Docker Desktop from Applications
# Wait until Docker Desktop is running (check the menu bar icon)

# Verify Docker installation
docker --version
docker ps
```

**Important**: Make sure Docker Desktop is running before proceeding!

### Step 1.3: Install kubectl
```bash
# Install kubectl
brew install kubectl

# Verify installation
kubectl version --client
```

### Step 1.4: Install Minikube
```bash
# Install Minikube
brew install minikube

# Verify installation
minikube version
```

### Step 1.5: Install Helm
```bash
# Install Helm
brew install helm

# Verify installation
helm version
```

### Step 1.6: Install Additional Tools
```bash
# Install Git (if not installed)
brew install git

# Install jq (JSON processor - useful for working with kubectl)
brew install jq

# Install k9s (optional but highly recommended - Kubernetes CLI UI)
brew install k9s
```

---

## Part 2: Start Minikube Cluster

### Step 2.1: Start Minikube
```bash
# Start Minikube with sufficient resources
minikube start --cpus=4 --memory=8192 --disk-size=50g --driver=docker

# This will take a few minutes...
# You should see: "Done! kubectl is now configured to use "minikube" cluster"
```

### Step 2.2: Verify Minikube is Running
```bash
# Check Minikube status
minikube status

# Should show:
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured

# Verify kubectl can connect
kubectl get nodes

# Should show one node in Ready state
```

### Step 2.3: Enable Required Addons
```bash
# Enable Ingress controller
minikube addons enable ingress

# Enable Metrics Server (for monitoring)
minikube addons enable metrics-server

# Enable Registry (for local image storage)
minikube addons enable registry

# Verify addons are enabled
minikube addons list | grep enabled
```

### Step 2.4: Configure Docker to Use Minikube Registry
```bash
# Point your shell to Minikube's Docker daemon
eval $(minikube docker-env)

# Verify you're using Minikube's Docker
docker ps

# You should see Kubernetes system containers
```

**Important**: This command only affects your current terminal session. You'll need to run it in each new terminal where you want to build images.

---

## Part 3: Prepare Your Frontend Application

### Step 3.1: Navigate to Frontend Directory
```bash
cd ~/GitHub/frontend-app/frontend-app
```

### Step 3.2: Create Dockerfile
Create a file named `Dockerfile` in the `frontend-app/frontend-app` directory:

```dockerfile
# frontend-app/frontend-app/Dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build application
ENV NODE_ENV=production
RUN npm run build

# Production stage with Nginx
FROM nginx:1.25-alpine

# Copy custom nginx config
COPY nginx.conf /etc/nginx/nginx.conf

# Copy built assets from builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Add healthcheck
RUN apk add --no-cache curl

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD curl -f http://localhost/ || exit 1

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Step 3.3: Create nginx.conf
Create `nginx.conf` in the same directory:

```nginx
# frontend-app/frontend-app/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;
    gzip on;

    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

### Step 3.4: Create .dockerignore
```plaintext
# frontend-app/frontend-app/.dockerignore
node_modules
.git
.gitignore
dist
npm-debug.log*
.env.local
.DS_Store
README.md
```

---

## Part 4: Build Frontend Image with Docker (Simple Method)

### Step 4.1: Make Sure You're Using Minikube's Docker
```bash
# Navigate to frontend directory
cd ~/GitHub/frontend-app/frontend-app

# Use Minikube's Docker daemon
eval $(minikube docker-env)

# Verify
docker info | grep Name
# Should show: Name: minikube
```

### Step 4.2: Build the Image
```bash
# Build the frontend image directly in Minikube
docker build -t frontend-app:1.0.0 .

# This will take a few minutes...

# Verify the image was built
docker images | grep frontend-app
```

---

## Part 5: Create Kubernetes Manifests

### Step 5.1: Create Directory Structure
```bash
# Create k8s directory in your project root
cd ~/GitHub
mkdir -p k8s/frontend-app

cd k8s/frontend-app
```

### Step 5.2: Create Namespace
Create `namespace.yaml`:

```yaml
# k8s/frontend-app/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: accommodation-booking
```

### Step 5.3: Create Deployment
Create `deployment.yaml`:

```yaml
# k8s/frontend-app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: accommodation-booking
  labels:
    app: frontend-app
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
        version: v1
    spec:
      containers:
      - name: frontend-app
        image: frontend-app:1.0.0
        imagePullPolicy: IfNotPresent  # Important for local images!
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
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
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Step 5.4: Create Service
Create `service.yaml`:

```yaml
# k8s/frontend-app/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-app
  namespace: accommodation-booking
  labels:
    app: frontend-app
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: frontend-app
```

### Step 5.5: Create Ingress
Create `ingress.yaml`:

```yaml
# k8s/frontend-app/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-app-ingress
  namespace: accommodation-booking
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: frontend.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-app
            port:
              number: 80
```

---

## Part 6: Deploy to Kubernetes

### Step 6.1: Apply the Manifests
```bash
# Make sure you're in the k8s/frontend-app directory
cd ~/GitHub/k8s/frontend-app

# Create namespace
kubectl apply -f namespace.yaml

# Deploy the application
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

### Step 6.2: Verify Deployment
```bash
# Check if pods are running
kubectl get pods -n accommodation-booking

# You should see something like:
# NAME                            READY   STATUS    RESTARTS   AGE
# frontend-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# frontend-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

# Check service
kubectl get svc -n accommodation-booking

# Check ingress
kubectl get ingress -n accommodation-booking
```

### Step 6.3: View Logs (if needed)
```bash
# Get pod name
kubectl get pods -n accommodation-booking

# View logs
kubectl logs -n accommodation-booking <pod-name>

# Follow logs
kubectl logs -n accommodation-booking <pod-name> -f
```

---

## Part 7: Access Your Application

### Method 1: Using Minikube Tunnel (Recommended)
```bash
# In a new terminal, start Minikube tunnel
minikube tunnel

# Keep this terminal open!
# You may need to enter your password

# In another terminal, add entry to /etc/hosts
echo "127.0.0.1 frontend.local" | sudo tee -a /etc/hosts

# Now you can access your app at:
# http://frontend.local
```

### Method 2: Using Port Forward
```bash
# Get pod name
kubectl get pods -n accommodation-booking

# Forward port from pod to your local machine
kubectl port-forward -n accommodation-booking pod/<pod-name> 8080:80

# Access at: http://localhost:8080
```

### Method 3: Using Minikube Service
```bash
# This will open the service in your browser
minikube service frontend-app -n accommodation-booking
```

---

## Part 8: Monitoring and Debugging

### Check Pod Status
```bash
# Get detailed pod information
kubectl describe pod -n accommodation-booking <pod-name>

# Check events
kubectl get events -n accommodation-booking --sort-by='.lastTimestamp'
```

### Execute Commands in Pod
```bash
# Get a shell in the pod
kubectl exec -it -n accommodation-booking <pod-name> -- sh

# Inside the pod, you can:
# - Check nginx config: cat /etc/nginx/nginx.conf
# - Check files: ls /usr/share/nginx/html
# - Test nginx: nginx -t
```

### View Resource Usage
```bash
# Check resource usage
kubectl top pods -n accommodation-booking
kubectl top nodes
```

### Using K9s (Interactive CLI)
```bash
# Launch k9s
k9s

# Press 0 to see all namespaces
# Type :pods to view pods
# Type :svc to view services
# Type :ingress to view ingresses
# Press Ctrl+C to exit
```

---

## Part 9: Build with Kaniko (Advanced Method)

If you want to use Kaniko for building images inside Kubernetes:

### Step 9.1: Create Docker Registry Secret
```bash
# If using Docker Hub
kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_DOCKERHUB_USERNAME \
  --docker-password=YOUR_DOCKERHUB_TOKEN \
  --docker-email=YOUR_EMAIL \
  -n accommodation-booking
```

### Step 9.2: Create Kaniko Build Job
Create `kaniko-build.yaml`:

```yaml
# k8s/frontend-app/kaniko-build.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kaniko-frontend-build
  namespace: accommodation-booking
spec:
  template:
    spec:
      containers:
      - name: kaniko
        image: gcr.io/kaniko-project/executor:latest
        args:
        - "--dockerfile=Dockerfile"
        - "--context=git://github.com/YOUR_USERNAME/frontend-app.git#main:frontend-app/frontend-app"
        - "--destination=YOUR_DOCKERHUB_USERNAME/frontend-app:1.0.0"
        - "--cache=true"
        volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker/
      restartPolicy: Never
      volumes:
      - name: docker-config
        secret:
          secretName: docker-credentials
          items:
          - key: .dockerconfigjson
            path: config.json
```

### Step 9.3: Run Kaniko Build
```bash
# Apply the Kaniko job
kubectl apply -f kaniko-build.yaml

# Watch the build progress
kubectl logs -f job/kaniko-frontend-build -n accommodation-booking

# After build completes, update deployment to use new image
kubectl set image deployment/frontend-app \
  frontend-app=YOUR_DOCKERHUB_USERNAME/frontend-app:1.0.0 \
  -n accommodation-booking
```

---

## Part 10: Cleanup (When Done Testing)

```bash
# Delete all resources in namespace
kubectl delete namespace accommodation-booking

# Or delete individual resources
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml
kubectl delete -f ingress.yaml

# Stop Minikube tunnel (Ctrl+C in that terminal)

# Stop Minikube (optional)
minikube stop

# Delete Minikube cluster (if you want to start fresh)
minikube delete
```

---

## Quick Reference Commands

```bash
# Start fresh
minikube start --cpus=4 --memory=8192
eval $(minikube docker-env)

# Build image
docker build -t frontend-app:1.0.0 .

# Deploy
kubectl apply -f k8s/frontend-app/

# Check status
kubectl get all -n accommodation-booking

# View logs
kubectl logs -f -n accommodation-booking -l app=frontend-app

# Access app
minikube tunnel  # In separate terminal
open http://frontend.local

# Cleanup
kubectl delete namespace accommodation-booking
```

---

## Troubleshooting

### Problem: Pod is in ImagePullBackOff
**Solution**: Make sure you used `eval $(minikube docker-env)` before building the image, and set `imagePullPolicy: IfNotPresent` in deployment.

### Problem: Can't access app via ingress
**Solution**: Make sure `minikube tunnel` is running and you added the entry to `/etc/hosts`.

### Problem: Minikube won't start
**Solution**: 
```bash
minikube delete
minikube start --cpus=4 --memory=8192 --driver=docker
```

### Problem: Docker build fails
**Solution**: Check if Docker Desktop is running and you have enough disk space.

### Problem: Port already in use
**Solution**:
```bash
# Find process using port 8080
lsof -ti:8080

# Kill the process
kill -9 $(lsof -ti:8080)
```

Now you're ready to deploy your frontend! Start with Part 1 and work through each section step by step.